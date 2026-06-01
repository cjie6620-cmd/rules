# ===== Spring Boot 后端规范（单体架构） =====

> 适用场景：单个 Spring Boot 应用，不涉及服务拆分、服务注册、网关等微服务概念
> 通用规范见 [backend-common.md](backend-common.md)

## 技术栈（必须遵守）

| 组件 | 版本 | 说明 |
|------|------|------|
| Java | 1.8 | 不用 17+ |
| Spring Boot | 2.7.18 | Java 8 最后稳定版 |
| MyBatis-Plus | 3.5.5 | 不用 4.0+（要 Java 17） |
| MySQL Connector | 8.0.33 | `mysql:mysql-connector-java`，不用 8.2.0+（改名且要 Java 11） |
| Druid | 1.2.21 | `druid-spring-boot-starter`，不用 `boot-3-starter` |
| Redis | Lettuce（BOM 管理） | `spring-boot-starter-data-redis` |
| Elasticsearch | 7.17.x（BOM 管理） | 按需引入，不用 8.x |
| Knife4j | 4.3.0 | `knife4j-openapi2-spring-boot-starter`，不用 `openapi3-jakarta` |
| MapStruct | 1.5.5.Final | 必须配合 `lombok-mapstruct-binding:0.2.0` |
| Lombok | 1.18.36 | |
| Hutool | 5.8.33 | 不用 6.x（要 Java 17） |
| Sa-Token | 1.37.0 | 认证授权，不用 1.38.0+（要 Java 17） |

> 单体架构不需要：Spring Cloud、Nacos、Sentinel、Gateway、RocketMQ

## 避坑清单

> 通用避坑项见 [backend-common.md § 通用避坑清单](backend-common.md#通用避坑清单)

## 开发命令
- `./mvnw spring-boot:run` — 启动应用（端口 8080）
- `./mvnw test` — 运行单元测试
- `./mvnw clean package` — 打包

## 项目结构（单体，按功能模块划分）

```
src/main/java/com/example/xxx
├── common/                      # 公共模块
│   ├── core/                    # R.java、BaseEntity、常量
│   ├── exception/               # GlobalExceptionHandler
│   ├── annotation/              # @Log 操作日志
│   └── utils/                   # 工具类
├── config/                      # SaTokenConfig、MybatisPlusConfig 等
├── modules/                     # 业务模块（按功能拆分，不是按层）
│   └── xxx/
│       ├── controller/          # Controller 层
│       ├── service/             # Service 接口
│       │   └── impl/            # Service 实现
│       ├── mapper/              # Mapper（MyBatis-Plus）
│       ├── domain/
│       │   ├── entity/          # 数据库实体
│       │   ├── dto/             # 入参对象
│       │   └── vo/              # 返回视图对象
│       └── enums/               # 业务枚举
└── Application.java         # 启动类
```

## 分层规范

> 详见 [backend-common.md § 一、分层规范](backend-common.md#一分层规范controller--service--mapper)

## 统一响应体 R[T]

> 详见 [backend-common.md § 二、统一响应体](backend-common.md#二统一响应体-rt)

## 全局异常处理

> 详见 [backend-common.md § 三、全局异常处理](backend-common.md#三全局异常处理)

## 参数校验规范

> 详见 [backend-common.md § 四、参数校验规范](backend-common.md#四参数校验规范)

## 代码注入规范

> 详见 [backend-common.md § 五、代码注入规范](backend-common.md#五代码注入规范)

## 安全规范
- 使用 Sa-Token 1.37.0（`sa-token-spring-boot-starter`，不用 `sa-token-spring-boot3-starter`）
- Token 存 Redis，支持登录/注销/权限校验/路由拦截
- 方法级权限：`@SaCheckPermission` / `@SaCheckRole`
- 敏感信息用环境变量，禁止明文配置
- 生产环境关闭 Knife4j

## 日志规范
- 使用 `@Slf4j`，禁止 `System.out.println`
- 操作日志：`@Log` 注解 + AOP（谁在什么时间做了什么）
- 异常日志：`log.error("描述", e)`，必须带完整堆栈
- 生产环境用 JSON 格式日志

## 事务规范

> 通用事务规则（基本规则、事务粒度控制、失效陷阱）见 [backend-common.md § 六、事务规范](backend-common.md#六事务规范通用部分)

## 接口规范
- RESTful：`GET /api/users/{id}`、`POST`、`PUT`、`DELETE`
- 分页：`pageNum` + `pageSize`，返回 `PageResult<T>`（含 total、list）
- 批量：`POST /api/users/batch`

## Knife4j 接口文档规范

> 通用文档规范（注解使用、字段描述写作标准、规则清单）见 [backend-common.md § 七、Knife4j 接口文档规范](backend-common.md#七knife4j-接口文档规范)

### 单体特有配置
- 访问地址：`http://localhost:8080/doc.html`

## 配置规范
- `application.yml` — 公共配置
- `application-dev.yml` — 本地开发
- `application-prod.yml` — 生产环境
- 敏感信息用 `${ENV_VAR}` 注入
- **必须加**：`spring.mvc.pathmatch.matching-strategy: ant_path_matcher`

## 测试规范

> 通用测试规范（设计方法、AAA 模式、场景清单）见 [backend-common.md § 八、测试规范](backend-common.md#八测试规范)
