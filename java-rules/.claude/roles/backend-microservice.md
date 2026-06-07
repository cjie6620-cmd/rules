# Spring Boot 后端规范（微服务架构）

> 适用场景：多服务拆分、需要服务注册/发现、网关、分布式事务、消息队列等
> 通用规范见 [backend-common.md](backend-common.md)

## 技术栈（必须遵守）

| 组件 | 版本 | 说明 |
|------|------|------|
| Java | 1.8 | 不用 17+ |
| Spring Boot | 2.7.18 | Java 8 最后稳定版 |
| Spring Cloud | 2021.0.5 | 与 Boot 2.7 对齐 |
| Spring Cloud Alibaba | 2021.0.5.0 | Nacos + Sentinel |
| Nacos Server | 2.2.3 | 服务注册 / 配置中心 |
| MyBatis-Plus | 3.5.5 | 不用 4.0+（要 Java 17） |
| MySQL Connector | 8.0.33 | `mysql:mysql-connector-java`，不用 8.2.0+（改名且要 Java 11） |
| Druid | 1.2.21 | `druid-spring-boot-starter`，不用 `boot-3-starter` |
| Redis | Lettuce（BOM 管理） | `spring-boot-starter-data-redis` |
| Elasticsearch | 7.17.x（BOM 管理） | 按需引入，不用 8.x |
| XXL-Job | 2.4.1 | 分布式任务调度 |
| RocketMQ | 2.2.3 | 不用 2.3.0+（偏向 Boot 3） |
| Knife4j | 4.3.0 | `knife4j-openapi2-spring-boot-starter`，不用 `openapi3-jakarta` |
| MapStruct | 1.5.5.Final | 必须配合 `lombok-mapstruct-binding:0.2.0` |
| Lombok | 1.18.36 | |
| Hutool | 5.8.33 | 不用 6.x（要 Java 17） |
| Sa-Token | 1.37.0 | 认证授权，不用 1.38.0+（要 Java 17） |
| Sentinel | 1.8.6（BOM 管理） | 限流 / 熔断 |

## 避坑清单

> 通用避坑项见 [backend-common.md § 通用避坑清单](backend-common.md#通用避坑清单)

- Nacos Server 版本必须与客户端对齐（2.2.3），版本不匹配会注册失败

## 开发命令
- `./mvnw spring-boot:run` — 启动单个服务
- `./mvnw test` — 运行单元测试
- `./mvnw clean package` — 打包
- `docker compose -f docker/docker-compose.yml up -d` — 启动 Nacos / MySQL / Redis 等基础设施

## 项目结构（多模块微服务）

```
xxx/                             # 根项目（聚合）
├── xxx-gateway/                 # 网关服务（Spring Cloud Gateway）
├── xxx-auth/                    # 认证服务（Sa-Token + OAuth2）
├── xxx-system/                  # 系统管理服务（用户、角色、菜单）
├── xxx-activity/                # 活动管理服务（业务核心）
├── xxx-common/                  # 公共模块（被所有服务依赖）
│   ├── xxx-common-core/         # R.java、BaseEntity、常量、工具类
│   ├── xxx-common-redis/        # Redis 配置与工具
│   ├── xxx-common-mybatis/      # MyBatis-Plus 公共配置
│   ├── xxx-common-security/     # Sa-Token 公共配置
│   └── xxx-common-log/          # @Log 操作日志
└── pom.xml                      # 父 POM（统一版本管理）
```

### 单个服务内部结构
```
xxx-system/
├── src/main/java/.../system/
│   ├── controller/              # Controller 层
│   ├── service/                 # Service 接口
│   │   └── impl/                # Service 实现
│   ├── mapper/                  # Mapper（MyBatis-Plus）
│   ├── domain/
│   │   ├── entity/              # 数据库实体
│   │   ├── dto/                 # 入参对象
│   │   └── vo/                  # 返回视图对象
│   ├── feign/                   # Feign 调用其他服务（可选）
│   └── enums/                   # 业务枚举
└── src/main/resources/
    ├── application.yml
    ├── bootstrap.yml            # Nacos 配置中心
    └── mapper/                  # MyBatis XML
```

## 服务间调用规范
- 同步调用：OpenFeign（`@FeignClient`）
- 异步通信：RocketMQ（解耦、削峰）
- Feign 接口定义放在调用方，返回值用 `R<T>` 包装
- Feign 降级必须配置 Sentinel fallback

## 分布式事务
- 强一致性：Seata AT 模式（小事务，跨 2~3 个服务）
- 最终一致性：RocketMQ 事务消息（大流量场景）
- 禁止跨服务直接调数据库，必须走 Feign 或 MQ

## 分层规范

> 详见 [backend-common.md § 一、分层规范](backend-common.md#一分层规范controller--service--mapper)

## 统一响应体 R[T]

> 详见 [backend-common.md § 二、统一响应体](backend-common.md#二统一响应体-rt)

## 全局异常处理（放在 xxx-common-core）

> 详见 [backend-common.md § 三、全局异常处理](backend-common.md#三全局异常处理)

## 参数校验规范

> 详见 [backend-common.md § 四、参数校验规范](backend-common.md#四参数校验规范)
> 自定义校验注解放 `xxx-common-core/annotation/`

## 代码注入规范

> 详见 [backend-common.md § 五、代码注入规范](backend-common.md#五代码注入规范)

## 安全规范
- 使用 Sa-Token 1.37.0（`sa-token-spring-boot-starter`，不用 `sa-token-spring-boot3-starter`）
- 网关统一鉴权（`SaReactorFilter`），业务服务只做权限校验
- Token 存 Redis，支持登录/注销/权限校验/路由拦截
- 方法级权限：`@SaCheckPermission` / `@SaCheckRole`
- 敏感信息用 Nacos 配置中心或环境变量，禁止硬编码
- 生产环境关闭 Knife4j

## 网关规范（xxx-gateway）
- 路由转发：按服务名自动路由（`lb://xxx-system/api/**`）
- 统一鉴权：`SaReactorFilter` 在网关层拦截
- 限流：Sentinel 网关限流规则
- 跨域：网关统一配置 CORS，业务服务不重复配置
- 日志：记录所有请求的 traceId，方便跨服务排查

## 日志规范
- 使用 `@Slf4j`，禁止 `System.out.println`
- 操作日志：`@Log` 注解 + AOP（谁在什么时间做了什么）
- 异常日志：`log.error("描述", e)`，必须带完整堆栈
- 生产环境用 JSON 格式日志
- **分布式日志**：所有请求带 traceId（Sleuth/Zipkin），跨服务可追踪

## 事务规范

> 通用事务规则（基本规则、事务粒度控制、失效陷阱）见 [backend-common.md § 六、事务规范](backend-common.md#六事务规范通用部分)

- 跨服务事务用 Seata AT 或 RocketMQ 事务消息

## 接口规范
- RESTful：`GET /api/users/{id}`、`POST`、`PUT`、`DELETE`
- 分页：`pageNum` + `pageSize`，返回 `PageResult<T>`（含 total、list）
- 批量：`POST /api/users/batch`
- 服务内部调用：Feign 接口，返回 `R<T>`

## 配置规范
- 公共配置放 Nacos 配置中心（`bootstrap.yml` 指定 `server-addr`）
- `application.yml` — 服务自身配置
- 敏感信息用 Nacos 加密配置或环境变量
- **必须加**：`spring.mvc.pathmatch.matching-strategy: ant_path_matcher`
- 每个服务配置 `spring.application.name`，与 Nacos 注册名一致

## Knife4j 接口文档规范

> 通用文档规范（注解使用、字段描述写作标准、规则清单）见 [backend-common.md § 七、Knife4j 接口文档规范](backend-common.md#七knife4j-接口文档规范)

### 微服务特有配置
- 访问地址：`http://localhost:{port}/doc.html`（每个服务独立端口）

## 测试规范

> 通用测试规范（设计方法、AAA 模式、场景清单）见 [backend-common.md § 八、测试规范](backend-common.md#八测试规范)

### 微服务特有测试项
- Feign 调用超时 / 降级生效
- MQ 消息发送与消费链路验证

### 微服务错误推测补充
- **服务调用超时 / 降级**
- **MQ 消息丢失 / 重复消费**

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
