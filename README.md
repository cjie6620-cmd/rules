# rules

Claude Code 开发规范集合，按技术栈拆分为独立的 CLAUDE.md 规则包，拷贝到项目根目录即可让 Claude Code 按规范生成代码。

两套规则互不依赖，按项目技术栈选用一套即可。

---

## 目录

- [java-rules](#java-rules) — Spring Boot + Vue3 全栈规范
- [python-rules](#python-rules) — Python LLM 应用规范（RAG + Agent）
- [使用方式](#使用方式)
- [自定义与占位符](#自定义与占位符)
- [版本维护](#版本维护)

---

## java-rules

适用于 **Spring Boot + Vue3 前后端分离**项目，支持单体和微服务两种架构切换。

### 技术栈

| 域 | 技术 | 版本 |
|----|------|------|
| 前端 | Vue3 + Vite + Ant Design Vue + Pinia | — |
| 后端 | Spring Boot + Java 8 + MyBatis-Plus | 2.7.18 / 3.5.5 |
| 数据库 | MySQL + Redis（Lettuce）+ Elasticsearch | 8.0.33 / 7.17.x |
| 微服务 | Spring Cloud + Spring Cloud Alibaba（Nacos + Sentinel） | 2021.0.5 |
| 中间件 | XXL-Job + RocketMQ | 2.4.1 / 2.2.3 |
| 工具 | Druid + Hutool + MapStruct + Knife4j + Sa-Token | 各见 CLAUDE.md |

### 架构选择

项目初始化时二选一，选定后对应规范全局生效：

| 架构 | 说明 | 对应角色文件 |
|------|------|------------|
| **单体** | Spring Boot + Sa-Token + Redis + MySQL | `backend-monolith.md` |
| **微服务** | 上述 + Nacos + Gateway + Seata + RocketMQ | `backend-microservice.md` |

### 角色文件一览（18 个）

| 角色 | 文件 | 覆盖内容 |
|------|------|---------|
| Frontend | `frontend.md` | Vue3 组件、路由、Pinia、Ant Design Vue、API 调用、SSE 消息渲染 |
| Backend-单体 | `backend-monolith.md` | Controller → Service → Mapper 分层、Sa-Token 鉴权、统一返回 |
| Backend-微服务 | `backend-microservice.md` | Feign 调用、分布式事务、网关鉴权、链路追踪 |
| Backend-模式 | `backend-patterns.md` | 设计模式、策略/责任链/Spring Events、Client 封装 |
| Backend-缓存 | `backend-cache.md` | Redis 缓存策略、穿透/击穿/雪崩、@Cacheable、本地缓存 |
| Backend-异步 | `backend-async.md` | 线程池、@Async、CompletableFuture、XXL-Job、@Scheduled |
| Backend-MQ | `backend-mq.md` | RocketMQ 生产/消费、死信队列、事务消息 |
| Backend-ES | `backend-elasticsearch.md` | ES 索引设计、查询 DSL、分词器、同步策略 |
| Backend-AOP | `backend-aop.md` | 自定义注解、操作日志切面、幂等切面、限流切面 |
| Backend-Java特性 | `backend-java-features.md` | Stream API、Optional、LocalDateTime、虚拟线程、i18n |
| Backend-文件 | `backend-file.md` | 文件上传/下载、OSS/MinIO、数据权限、多租户 |
| DBA | `dba.md` | 建表规范、索引设计、数据迁移、字段注释 |
| DBA-优化 | `dba-optimization.md` | SQL 优化、EXPLAIN 分析、慢查询治理、分页优化 |
| DevOps | `devops.md` | Docker Compose、Nginx、CI/CD、部署脚本 |
| Reviewer | `reviewer.md` | Code Review 检查清单、质量门禁 |
| AI-Spring | `ai-spring.md` | Spring AI 集成规范 |
| 全局 | `CLAUDE.md` | 命名规范、注释规范、Git 规范、验证规则、角色自动加载映射 |

### 核心规范摘要

- **命名**：Java 类 `PascalCase`，方法/变量 `camelCase`，数据库表/字段 `snake_case`，常量 `UPPER_SNAKE_CASE`
- **注释**：禁止无意义注释，只注释 WHY；公共方法必须 Javadoc；VO/DTO 必须加 Knife4j `@ApiModelProperty`
- **分层**：Controller（接收请求）→ Service（业务逻辑）→ Mapper（数据访问）→ Entity/DTO/VO（数据对象）
- **API 返回**：统一 `Result<T>` 包装，禁止裸返回
- **禁止**：硬编码魔法值、`any` 类型（TS 端）
- **验证**：每次改动必须执行 `pnpm build`（前端）或 `mvnw clean install`（后端），编译失败禁止交付
- **Git**：提交格式 `feat(scope): 描述`，分支命名 `feat/xxx`、`fix/xxx`

---

## python-rules

适用于 **FastAPI / Flask + RAG + Agent** 项目，覆盖 LLM 应用从原型到生产的全流程。

### 技术栈

| 域 | 技术 | 版本/说明 |
|----|------|----------|
| 语言 | Python | 3.12+ |
| 包管理 | uv | latest |
| Web 框架 | FastAPI / Flask | 0.136+ / 3.1+ |
| 数据校验 | Pydantic | v2 |
| ORM | SQLAlchemy（async） | 2.0 |
| 迁移 | Alembic | latest |
| 向量库 | Milvus / Qdrant / PGVector | 视场景 |
| LLM 框架 | LangChain / LlamaIndex / LangGraph | 1.0+ |
| LLM 主力 | **DeepSeek**（deepseek-chat / deepseek-reasoner） | langchain-deepseek 0.1+ |
| LLM 备选 | 通义千问 / 智谱 GLM / 零一万物 / 豆包 | 视场景 |
| Agent 框架 | LangGraph（生产）/ smolagents（轻量） | latest |
| 可观测 | Langfuse | 2.x |
| Guardrails | Guardrails AI | latest |
| 前端 | Vue 3.4+ / Vite 5+ / Pinia / Ant Design Vue | latest |
| 数据库 | PostgreSQL 16+（with pgvector） | 16+ |
| 缓存 | Redis | 7+ |
| 对象存储 | MinIO | latest |
| 部署 | Docker Compose + GPU 编排 | latest |

> **模型合规约束**：所有 LLM 调用必须使用国产大模型，禁止使用 OpenAI / Anthropic / Cohere 等国外 API。

### RAG 项目架构

```
用户交互层（SSE）
    ↓
意图识别（LLM）→ 不相关 → 通用对话
                → 相关 → 查询改写 → 查询路由
                                      ↓
                              文档检索 / 结构化查询 / 图数据库
                                      ↓
                              Reranking + 聚合
                                      ↓
                              LLM 流式生成 + 引用溯源 + 结构化卡片
```

文档入库流程：`文件上传 → 格式转换 → 智能切片 → 向量化嵌入 → 索引存储`

### 角色文件一览（15 个）

| 角色 | 文件 | 覆盖内容 |
|------|------|---------|
| Frontend | `frontend.md` | AI 聊天 UI、SSE 消息解析、Markdown 渲染、Pinia store |
| Backend-FastAPI | `backend-fastapi.md` | 路由、Pydantic Schema、依赖注入、async Service 层 |
| Backend-Flask | `backend-flask.md` | Blueprint、marshmallow Schema、Celery 异步任务 |
| DBA | `dba.md` | SQLAlchemy Model、Alembic 迁移、PGVector 列、seed 脚本 |
| DevOps | `devops.md` | Docker Compose、vLLM 部署、GPU 编排、Prometheus + Grafana 监控 |
| Reviewer | `reviewer.md` | PR Review 检查清单、CI 门禁 |
| **Agent（核心）** | `agent.md` | RAG 检索、Agent Loop、Prompt 模板、向量库、Memory、LLM Ops |
| Agent-RAG | `agent-rag.md` | 混合检索、Re-ranking、VectorStore 设计 |
| Agent-LLMOps | `agent-llmops.md` | Memory 策略、Langfuse 可观测、Guardrails、失败回退 |
| Agent-Cost | `agent-cost.md` | Token 预算、成本监控、评估流水线、延迟优化 |
| Prompt Engineering | `prompt-engineering.md` | 10 条设计原则、10 种 Prompt 模式、Few-shot、RAG Prompt 优化 |
| MCP | `mcp.md` | MCP Server 开发、Tool 设计、MCP 与 Agent 集成 |
| Skill | `skill.md` | Skill 创建、渐进式披露、Action-First 原则、SKILL.md 模板 |
| RAG Evaluation | `rag-evaluation.md` | RAGAS 指标、评测流水线、CI 回归门禁、golden dataset 设计 |
| Security | `security.md` | API Key 管理、SQL 注入防护、Prompt 注入防护、JWT、CORS |

### 核心规范摘要

- **异常体系**：统一基类，分层映射 HTTP 状态码（BizException 400、NotFoundException 404、LLMException 503 等）
- **LLM 重试**：tenacity 指数退避，只重试可恢复错误（RateLimit / Timeout / Connection）
- **降级策略**：主力模型不可用切备用模型，向量库不可用降级全文检索
- **代码风格**：ruff（PEP 8 行宽 120）+ mypy strict，CI 必须通过
- **测试**：单元测试 Service 层 80%+，集成测试 API 60%+；禁止测试中调真实 LLM，必须 mock
- **Prompt 管理**：以文件存储（`.txt` / `.jinja2`），禁止硬编码在代码中，Jinja2 模板注入变量
- **安全**：API Key 走 `.env`，密码 bcrypt 哈希，SQL 参数化查询，Prompt 输入消毒
- **Git**：Conventional Commits，`main` 保护分支，PR 必须 CI 通过 + 1 人 approve

### 四大新增角色的协作关系

```
Prompt Engineering ←→ Agent（Prompt 模板版本管理）
        ↓
MCP Server ←→ Agent（Tool 暴露和调用）
        ↓
Skill ←→ Agent（领域知识编排）
        ↓
RAG Evaluation ←→ Agent（质量保障和迭代）
```

- **Prompt Engineering** 是基础：好的 Prompt 是 RAG 和 Agent 效果的前提
- **MCP** 是 Tool 层标准化：让 Tool 生态可复用、可组合
- **Skill** 是知识层标准化：让 Agent 领域知识模块化、可维护
- **RAG Evaluation** 是质量保障：用数据驱动迭代，避免"改了就上线"

---

## 使用方式

### 1. 拷贝规则到项目

```bash
# Java 项目
cp -r java-rules/CLAUDE.md your-project/CLAUDE.md
cp -r java-rules/.claude your-project/.claude

# Python 项目
cp -r python-rules/CLAUDE.md your-project/CLAUDE.md
cp -r python-rules/.claude your-project/.claude
```

### 2. 工作原理

Claude Code 启动时自动读取项目根目录的 `CLAUDE.md`，并根据任务内容从 `.claude/roles/` 中加载对应角色文件，按规范约束生成代码，不需要手动指定角色。

角色自动加载规则：

- 修改 `.vue` / `.ts` 文件 → 自动加载 `frontend.md`
- 修改 `.java` 文件 → 自动加载对应后端规范（单体或微服务）
- 涉及缓存 / MQ / ES / AOP → 同时加载对应专项规范
- 涉及 RAG / Agent / Prompt → 自动加载 `agent.md`（python-rules 核心）
- 用户说"审查代码" / "review" → 自动加载 `reviewer.md`

### 3. 自定义

- `CLAUDE.md` 中的 `xxx` 均为占位符，替换为实际业务模块名即可
- 项目初始化时需确定架构（单体/微服务），选定后对应规范全局生效
- 用户也可手动指定角色："用前端角色"、"用后端角色"，手动指定优先级最高

---

## 版本维护

两套规则均内置版本核实机制：

- **复核周期**：每季度（3 / 6 / 9 / 12 月）
- **核查工具**：Maven Central（Java）、Context7 MCP 工具（Python）
- **版本落后分级**：patch 低优先级 → minor 评估 breaking change → major 立即更新
- python-rules 可运行 `python scripts/check_versions.py` 自动扫描版本号
