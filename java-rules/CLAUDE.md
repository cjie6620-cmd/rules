# Java 后端 + Vue3 前端全栈开发规范

> 前后端分离架构，通用开发规范，不绑定特定业务项目。`xxx` 为占位符，使用时替换为实际业务模块名。
>
> 本文件只承担**目录索引 + 全局铁律**。所有详细规范、代码示例、测试模板、错误清单均下沉到 `.claude/roles/` 下的具体角色文件。

---

## 一、技术栈速查

| 域 | 技术 | 版本 |
|----|------|------|
| 前端 | Vue3 + Vite + Ant Design Vue | latest |
| 后端语言 | Java | 8 |
| 后端框架 | Spring Boot | 2.7.18 |
| ORM | MyBatis-Plus | 3.5.5 |
| 数据库 | MySQL | 8.0.33 |
| 缓存 | Redis（Lettuce） | — |
| 搜索引擎 | Elasticsearch | 7.17.x |
| 微服务（可选） | Spring Cloud + Alibaba（Nacos + Sentinel） | 2021.0.5 / 2021.0.5.0 |
| 中间件 | XXL-Job / RocketMQ | 2.4.1 / 2.2.3 |
| 工具 | Druid / Hutool / MapStruct / Knife4j / Sa-Token | 1.2.21 / 5.8.33 / 1.5.5 / 4.3.0 / 1.37.0 |

> 详细版本核实机制：见 `roles/reviewer.md` 末尾「版本核实机制」。

---

## 二、目录结构

```
project/
├── frontend/    # Vue3 前端
├── backend/     # Spring Boot 后端
├── db/          # 数据库脚本
└── scripts/     # 构建/部署脚本
```

---

## 三、角色索引（任务特征 → 自动加载 role.md）

> **核心原则**：AI 收到任务后，**必须**先看本表 → 自动加载对应 role.md → 写代码前再写。
>
> 一个任务可同时加载多个角色（数据流方向：设计 → 实现 → 数据 → 测试 → 部署 → 审查）。

| 任务特征 | 加载角色 | 关键产出 |
|---------|---------|---------|
| 修改 `frontend/`、`.vue` / `.ts`、Vue 组件 / 路由 / 样式 / API 调用 | [frontend.md](.claude/roles/frontend.md) | Vue3 + Ant Design Vue 组件 / 页面 |
| 后端单体（Controller / Service / Mapper / 业务逻辑） | [backend-monolith.md](.claude/roles/backend-monolith.md) | Spring Boot 单体项目分层 |
| 后端微服务（加 Nacos / Sentinel / Gateway / RocketMQ） | [backend-microservice.md](.claude/roles/backend-microservice.md) | 微服务拆分 / 服务间调用 / 分布式事务 |
| 通用后端模式（Service 编排 / 业务工具 / 公共基类） | [backend-common.md](.claude/roles/backend-common.md) | 业务公共代码 |
| 设计模式（策略 / 责任链 / Spring Events / Client 封装） | [backend-patterns.md](.claude/roles/backend-patterns.md) | 模式选型 + 落地代码 |
| 缓存策略（Redis / 缓存穿透 / 击穿 / 雪崩 / @Cacheable） | [backend-cache.md](.claude/roles/backend-cache.md) | 缓存方案 |
| 异步任务（线程池 / @Async / CompletableFuture / XXL-Job / @Scheduled） | [backend-async.md](.claude/roles/backend-async.md) | 异步任务模板 |
| 消息队列（RocketMQ / 事务消息 / 死信） | [backend-mq.md](.claude/roles/backend-mq.md) | MQ 方案 |
| 全文检索（Elasticsearch / 索引设计 / 分词器） | [backend-elasticsearch.md](.claude/roles/backend-elasticsearch.md) | ES 集成 |
| AOP 切面（自定义注解 / 鉴权 / 日志 / 限流 / 幂等） | [backend-aop.md](.claude/roles/backend-aop.md) | 切面模板 |
| Java 高级特性（Stream / Optional / 21 新特性 / 虚拟线程 / i18n） | [backend-java-features.md](.claude/roles/backend-java-features.md) | Java 语言特性 |
| 文件存储（上传下载 / OSS / MinIO / 多租户） | [backend-file.md](.claude/roles/backend-file.md) | 文件方案 |
| SQL 优化（EXPLAIN / 索引 / 慢查询 / 分页） | [dba-optimization.md](.claude/roles/dba-optimization.md) + [dba.md](.claude/roles/dba.md) | 性能调优 |
| 数据库脚本（`db/` / `.sql` / 建表 / 索引 / 数据迁移） | [dba.md](.claude/roles/dba.md) | DDL + 种子数据 |
| 部署（`scripts/` / docker-compose / CI / Nginx） | [devops.md](.claude/roles/devops.md) | 部署脚本 |
| 代码审查 / Review / 质量检查 | [reviewer.md](.claude/roles/reviewer.md) | 审查清单 + 输出格式 |
| **接口集成测试**（Testcontainers / *ApiTest / 8 类用例） | [integration-test.md](.claude/roles/integration-test.md) | 真实 DB 测试模板 |
| Spring AI / 智能体 / Function Calling / MCP | [ai-spring.md](.claude/roles/ai-spring.md) | AI 集成 |

> **手动指定优先级最高**：用户说"用前端角色" / "用后端角色"时，绕过自动判断。

---

## 四、架构选择（二选一，项目初始化时确定）

| 架构 | 触发条件 | 后端规范 | 不涉及 |
|------|---------|---------|--------|
| **单体** | 用户明确选单体 / 项目体量小 | `backend-monolith.md` + `backend-common.md` | Nacos / Sentinel / Gateway / RocketMQ / 分布式事务 |
| **微服务** | 用户明确选微服务 / 项目有服务拆分 | `backend-microservice.md` + `backend-common.md` | — |

微服务额外涉及：Feign 调用、Seata/MQ 分布式事务、网关鉴权、链路追踪。

---

## 五、通用铁律（所有角色都遵守）

### 5.1 命名

| 类型 | 规则 | 示例 |
|------|------|------|
| JS/TS 变量函数 | camelCase | `getUserInfo` |
| Vue 组件 | PascalCase | `UserProfile.vue` |
| Java 类 | PascalCase | `UserService` |
| Java 方法 | camelCase | `findById` |
| 数据库表/字段 | snake_case | `user_info`、`create_time` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Java 包路径 | `{base-package}.{domain}` | `com.example.activity` |
| 前端 views | 按业务模块分文件夹 | `views/xxx/` |
| 组件命名 | `{业务名}{组件类型}` | `XxxList.vue`、`XxxForm.vue` |
| 数据库表前缀 | `{模块缩写}_` | `xxx_activity` |

> **业务模块名标准**：看名字就知道做什么业务。`activity` / `member` / `finance` 是好的；`module1` / `sys_mgr` / `data` 是禁止的。
> **拿不准时**：把名字给一个不看代码的产品经理看，能直接猜出业务就算合格。

### 5.2 注释

- **禁止**无意义注释（`// 获取用户` ← 不要写）
- 只注释 **WHY**（为什么这么做），不注释 WHAT
- Java 公共方法必写 Javadoc（一句话 + `@param` / `@return`）
- 业务逻辑长时用 `// 1. 校验参数` `// 2. 检查库存` 标注主流程
- VO/DTO/Entity 字段必加 Knife4j `@ApiModelProperty`，value 写含义+约束，example 填真实业务数据
- Entity 字段 Javadoc 必须说明字段含义、枚举值、约束（如"0-正常 1-停用"），与 DB 注释一致

### 5.3 禁止项

- ❌ 任何 `any` 类型
- ❌ API 返回不统一用 `Result<T>` 包装
- ❌ 硬编码魔法数字 / 字符串（抽到常量）
- ❌ 单元测试 Mock Mapper / 跳过真实 DB
- ❌ 接口变更不写 `*ApiTest` 8 类用例
- ❌ 越权访问返 404（必须 403，避免信息泄露）
- ❌ 测试方法不带 `@DisplayName`（Allure 无法定位失败）
- ❌ MySQL 镜像用 `latest`（版本漂移 → "本地能跑 CI 挂"）

---

## 六、命令速查

### 前端（跨平台）

```bash
pnpm dev / build / lint
```

### 后端（按 Shell 选择）

| Shell | 命令 |
|-------|------|
| Git Bash / MSYS / WSL | `./mvnw clean install` / `./mvnw spring-boot:run` / `./mvnw test` |
| cmd / PowerShell | `mvnw.cmd clean install` / `mvnw.cmd spring-boot:run` / `mvnw.cmd test` |

### 接口集成测试

```bash
mvnw test -Dtest='*ApiTest'                        # 全部
mvnw test -Dtest='LoginApiTest'                    # 单端点
mvnw test -Dtest='LoginApiTest#login_locked_after_5_failures'  # 单方法
```

> 当前环境 Windows（`{{.OS_TYPE}}`），按实际 Shell 选择。详细 Testcontainers 配置、8 类用例模板、CI 接入 → [integration-test.md](.claude/roles/integration-test.md)。

---

## 七、代码改动后验证流程（强制）

```
1. 改代码
2. 前端改 → pnpm build
   后端改 → ./mvnw clean install
3. 涉及接口 → 跑 mvnw test -Dtest='*ApiTest'
4. 需要中间件（MySQL/Redis/ES）→ docker compose -f docker/docker-compose.yml up -d
5. 端口检查：netstat -ano | findstr ":8080"（被占则杀旧进程或问用户）
6. 启动顺序：中间件 → 后端 → 前端 → 测试接口
```

详细端口处理、Docker 修复、测试用例原则 → [devops.md](.claude/roles/devops.md) + [integration-test.md](.claude/roles/integration-test.md)。

---

## 八、README 维护规则

- 项目根目录必须有 `README.md`
- **任何代码改动**（新增功能 / 改接口 / 改配置 / 改依赖）必须同步更新 README
- README 必含：项目简介 / 技术栈 / 启动方式 / 目录结构 / 接口文档入口