# 通用 Java 后端 + Vue3 前端全栈开发规范

前后端分离架构，通用开发规范，不绑定特定业务项目。`xxx` 为占位符，使用时替换为实际业务模块名。

## 技术栈
- 前端：Vue3 + Vite + Ant Design Vue
- 后端：Spring Boot 2.7.18 + Java 8 + MyBatis-Plus 3.5.5
- 数据库：MySQL 8.0.33 + Redis（Lettuce）+ Elasticsearch 7.17.x
- 微服务：Spring Cloud 2021.0.5 + Spring Cloud Alibaba 2021.0.5.0（Nacos + Sentinel）
- 中间件：XXL-Job 2.4.1 + RocketMQ 2.2.3
- 工具：Druid 1.2.21 + Hutool 5.8.33 + MapStruct 1.5.5 + Knife4j 4.3.0 + Sa-Token 1.37.0

## 目录结构
```
project/
├── frontend/    # Vue3 前端
├── backend/     # Spring Boot 后端
├── db/          # 数据库脚本
└── scripts/     # 构建/部署脚本
```

## 全局命令
- 前端：`pnpm dev` / `pnpm build` / `pnpm lint`
- 后端：`./mvnw clean install` / `./mvnw spring-boot:run` / `./mvnw test`

## Git 规范
- 提交格式：`feat(scope): 描述`
- 分支命名：`feat/xxx`、`fix/xxx`、`chore/xxx`

## 注释与命名规范（全局生效）

### 命名
- JS/TS 变量函数：camelCase（`getUserInfo`）
- Vue 组件：PascalCase（`UserProfile.vue`）
- Java 类名：PascalCase（`UserService`）
- Java 方法：camelCase（`findById`）
- 数据库表/字段：snake_case（`user_info`、`create_time`）
- 常量：UPPER_SNAKE_CASE（`MAX_RETRY_COUNT`）

### 模块与包命名（按实际业务语义命名，用通俗易懂的英文）

**原则：看名字就知道是做什么业务，不需要猜。**

- Java 包路径：`{base-package}.{domain}`，如 `com.example.activity`、`com.example.member`
- 前端 views 目录：按业务模块分文件夹，如 `views/xxx/`
- 组件命名：`{业务名}{组件类型}`，如 `XxxList.vue`、`XxxForm.vue`
- 数据库表前缀：`{模块缩写}_`，如 `xxx_activity`、`xxx_member`

**好的命名举例**（一看就懂）：
```
activity    # 活动管理
member      # 成员管理
finance     # 财务
notice      # 通知公告
checkin     # 签到打卡
feedback    # 反馈意见
```

**禁止的命名**（看不懂业务含义）：
```
module1 / module2    # 无意义序号
sys_mgr / sys_op     # 过度缩写
data / info / util   # 太泛化，不表达业务
```

**拿不准时的标准**：把这个模块名给一个不看代码的产品经理看，能直接猜出是什么业务就算合格。

### 注释
- 禁止无意义注释（`// 获取用户` ← 不要写）
- 只注释 WHY（为什么这么做），不注释 WHAT（做了什么）
- Java 公共方法必须写 Javadoc（一句话 + `@param` / `@return`）
- 业务逻辑较长时，用序号标注主流程步骤（如 `// 1. 校验参数` `// 2. 检查库存` `// 3. 创建订单`），方便快速定位流程节点
- VO/DTO/Entity 字段必须加 Knife4j 注解（`@ApiModelProperty`），value 写清楚字段含义和约束，example 填真实业务数据
- Entity 数据库实体字段必须加 Javadoc 注释，说明字段含义、枚举值、约束（如"0-正常 1-停用"），与数据库表注释保持一致

## 禁止事项
- 不允许使用 `any` 类型
- API 返回必须统一用 `Result<T>` 包装
- 禁止硬编码魔法数字和字符串

## README 维护规则
- 项目根目录必须有 `README.md`
- **任何代码改动（新增功能、改接口、改配置、改依赖）都必须同步更新 README.md**
- README 需要包含：项目简介、技术栈、启动方式、目录结构说明、接口文档入口

## 代码改动后验证规则（每次改动必须执行）

### 编译与测试
- **前端改动** → 执行 `pnpm build`，确认编译通过无报错
- **后端改动** → 执行 `./mvnw clean install`，确认编译通过无报错
- 前后端都有改动 → 两边都编译测试，先前端再后端
- 编译失败必须立即修复，不允许带编译错误交付

### 中间件与 Docker（以下场景必须先拉起中间件）
- 后端需要连接 MySQL / Redis / ES 才能启动或测试时
- 执行后端集成测试（需要真实数据库的测试）
- 本地联调前后端（前端请求后端接口，后端依赖中间件）
- **启动命令**：`docker compose -f docker/docker-compose.yml up -d`
- **校验命令**：`docker compose -f docker/docker-compose.yml ps`，确认三个服务都是 `healthy`
- 纯前端改动（不涉及后端）不需要启动 Docker
- Docker 未启动或容器异常时，按 `.claude/roles/devops.md` 中的修复流程处理

### API 测试前的端口检查（每次测试接口必须先执行）

**流程：检查端口 → 判断是否重启 → 启动服务 → 测试接口**

```bash
# 1. 检查后端默认端口（8080）是否已占用
netstat -ano | findstr ":8080"

# 2. 根据情况处理
#    - 是本项目的旧进程 → 杀掉重启（拿到最新代码）
#    - 是其他服务占用 → 询问用户是否换端口，或关掉对方
#    - 端口空闲 → 直接启动
```

**判断规则**：
- 端口空闲 → 直接启动后端，然后测试接口
- 端口被占用，且是本项目的旧 Java 进程（通过 `netstat -ano` + `tasklist /FI "PID eq xxx"` 确认）→ 杀掉旧进程，重新启动
- 端口被占用，但不是本项目的进程 → 询问用户，不要擅自杀进程
- 前端测试同理：检查 5173 端口，处理方式相同

**启动顺序**：中间件（Docker） → 后端 → 前端 → 测试接口

### 测试用例
- **大功能（新增模块、新增核心接口、复杂业务逻辑）** → 必须主动写单元测试
- **小功能（改字段、加个参数、修个 bug）** → 不主动写，但编译前询问用户"是否需要补充测试用例？"
- 后端测试放 `src/test/`，用 JUnit 5 + Mockito
- 前端测试放 `tests/`，用 Vitest（如已配置）

## 架构选择（根据项目实际情况二选一）

| 架构 | 技术栈 | 后端规范文件 |
|------|--------|-------------|
| **单体架构** | Spring Boot 2.7.18 + MyBatis-Plus + Sa-Token + Redis + MySQL | `.claude/roles/backend-monolith.md` |
| **微服务架构** | 上述 + Spring Cloud 2021.0.5 + Nacos + Sentinel + RocketMQ + XXL-Job + Gateway | `.claude/roles/backend-microservice.md` |

- 项目初始化时由用户确定架构，选定后对应的后端规范全局生效
- 单体不涉及：Nacos、Sentinel、Gateway、RocketMQ、分布式事务
- 微服务额外涉及：服务间调用（Feign）、分布式事务（Seata/MQ）、网关鉴权、链路追踪

## 角色自动加载规则

**根据任务内容自动读取对应规范文件，不需要用户手动指定。**

### 触发条件 → 加载文件

| 任务特征（出现任一即触发） | 加载的规范文件 |
|--------------------------|---------------|
| 修改 `frontend/` 目录、`.vue` / `.ts` / `.css` 文件、涉及 Vue 组件 / 路由 / 样式 / API 调用 | `.claude/roles/frontend.md` |
| 修改 `backend/` 目录、`.java` 文件、涉及 Controller / Service / Mapper / 接口开发 / 业务逻辑 | `.claude/roles/backend-monolith.md`（单体）或 `.claude/roles/backend-microservice.md`（微服务） |
| 涉及设计模式、工具类封装、Client封装、策略模式、责任链、Spring Events、外部HTTP调用封装 | `.claude/roles/backend-patterns.md`（同时加载对应的后端架构规范） |
| 涉及缓存策略、Redis 使用、缓存穿透/击穿/雪崩、@Cacheable、本地缓存 | `.claude/roles/backend-cache.md`（同时加载对应的后端架构规范） |
| 涉及线程池、@Async、CompletableFuture、定时任务、XXL-Job、@Scheduled | `.claude/roles/backend-async.md`（同时加载对应的后端架构规范） |
| 涉及 RocketMQ、消息队列、异步消息、死信队列、事务消息 | `.claude/roles/backend-mq.md`（同时加载对应的后端架构规范） |
| 涉及 Elasticsearch、ES 查询、全文检索、索引设计、分词器 | `.claude/roles/backend-elasticsearch.md`（同时加载对应的后端架构规范） |
| 涉及 AOP、切面编程、自定义注解、操作日志切面、幂等切面、限流切面 | `.claude/roles/backend-aop.md`（同时加载对应的后端架构规范） |
| 涉及 Stream API、Optional、LocalDateTime、Java 21、虚拟线程、国际化/i18n | `.claude/roles/backend-java-features.md` |
| 涉及文件上传/下载、OSS、MinIO、数据权限、多租户、数据隔离 | `.claude/roles/backend-file.md`（同时加载对应的后端架构规范） |
| 涉及 SQL 优化、EXPLAIN、索引设计、慢查询、分页优化 | `.claude/roles/dba-optimization.md`（同时加载 `.claude/roles/dba.md`） |
| 修改 `db/` 目录、`.sql` 文件、涉及建表 / 索引 / 查询优化 / 数据迁移 | `.claude/roles/dba.md` |
| 修改 `scripts/` 目录、`docker-compose.yml`、涉及部署 / Docker / CI / Nginx | `.claude/roles/devops.md` |
| 用户说"审查代码" / "review" / "看看这段代码"、或主动进行代码质量检查 | `.claude/roles/reviewer.md` |

### 执行规则

1. **判断任务涉及哪些角色**，可能同时触发多个（如前后端联调同时加载前端规范和后端规范）
2. **先读规范文件，再回答或写代码**，确保输出符合对应规范
3. **用户也可以手动指定**（"用前端角色" / "用后端角色"），手动指定优先级最高
4. **跨角色任务**按涉及的模块分别加载，每个模块遵守各自规范

## 版本核实机制（强制）

> 确保所有规范文件中的技术库版本不落后于官方最新版。

### 规则

1. 复核周期：每季度（3/6/9/12 月）
2. 核查工具：**Maven Central** + **Context7**
3. 核查对象：Spring Boot / Spring Cloud / MyBatis-Plus / Sa-Token / Knife4j / Hutool 等

### 核查流程

1. 查询各组件 Maven Central 最新版本
2. 对比规范中写的版本，落后 > 1 个小版本则标记为"需更新"
3. 有破坏性变更（major version bump）**必须**更新代码示例和避坑清单

### 版本落后分级

| 落后程度 | 处理 |
|---------|------|
| patch 版本落后 | 记录，低优先级 |
| minor 版本落后 | 评估 breaking change，中优先级 |
| major 版本落后（如 Spring Boot 2.x → 3.x）| **[严重]** 必须评估升级路径 |
