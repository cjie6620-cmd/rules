# Python LLM 应用开发规范

> FastAPI / Flask + RAG + Agent + LangChain/LlamaIndex 全栈项目。
>
> 本文件只承担**目录索引 + 全局铁律**。所有详细规范、代码示例、测试模板均下沉到 `.claude/roles/` 下的具体角色文件。

---

## 一、技术栈速查

| 域 | 技术 | 版本 |
|----|------|------|
| 语言 | Python | 3.12+ |
| 包管理 | uv | latest |
| Web 框架 | FastAPI / Flask | 0.136+ / 3.1+ |
| 数据校验 | Pydantic | v2 |
| ORM | SQLAlchemy | 2.0 (async) |
| 迁移 | Alembic | latest |
| 向量库 | Milvus / Qdrant / PGVector | 视场景 |
| LLM 框架 | LangChain / LlamaIndex / LangGraph | 1.0+ |
| **LLM 主力** | **DeepSeek**（deepseek-chat / deepseek-reasoner） | langchain-deepseek 0.1+ |
| LLM 备选 | 通义千问 / 智谱 GLM / 零一万物 / 豆包 | 视场景 |
| **LLM 国外 API** | **❌ 禁止**（OpenAI / Anthropic / Cohere / Jina / OpenRouter 全部禁用） | — |
| Agent 框架 | LangGraph（生产）/ smolagents（轻量） | latest |
| 可观测 | Langfuse | 2.x |
| Guardrails | Guardrails AI | latest |
| 前端 | Vue 3.4+ / Vite 5+ / Pinia / Ant Design Vue | latest |
| 数据库 | PostgreSQL 16+ (with pgvector) | 16+ |
| 缓存 | Redis 7+ | 7+ |
| 对象存储 | MinIO | latest |
| 消息队列 | Redis Stream / RabbitMQ / Kafka | 视场景 |
| 监控 | Prometheus + Grafana | latest |
| 部署 | Docker Compose + GPU 编排 | latest |

> 版本核实机制：见 `roles/reviewer.md` 末尾「版本核实机制」。所有 AI 角色文件顶部必须有「最后核对日期」。

---

## 二、RAG 项目通用架构

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          用户交互层 (SSE)                                │
│         {API Controller} · {Web UI} · {IM Bot}                          │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │
              ┌─────────────────▼──────────────────┐
              │          1. 意图识别                 │
              │   IntentRecognitionService (LLM)    │
              │   Structured Output → Record        │
              └────┬─────────────────────┬─────────┘
                   │ 不相关               │ 相关
           ┌───────▼───────┐    ┌────────▼────────────────────────────────┐
           │  通用对话      │    │           2. 查询改写                    │
           │  CommonChat   │    │  QueryTransformer (LLM)                │
           └───────┬───────┘    │  策略: 简洁/抽象/纠错/标准化              │
                   │            └────────┬─────────────────────────────────┘
                   │         ┌───────────▼────────────────────────────────┐
                   │         │           3. 查询路由                       │
                   │         │  QueryRouter (LLM)                        │
                   │         │  {intent, strategy, confidence} → 路由决策  │
                   │         └──┬────────────┬────────────┬──────────────┘
                   │            │            │            │
                   │    ┌───────▼──┐   ┌─────▼────┐  ┌──▼──────────┐
                   │    │ 文档检索  │   │ 结构化查询│  │  图数据库    │
                   │    └───────┬──┘   └─────┬────┘  └──┬──────────┘
                   │            │            │           │
                   │            └────────────┼───────────┘
                   │              ┌──────────▼──────────────────────────┐
                   │              │  4. Reranking + 聚合                  │
                   │              │  ContentAggregator                   │
                   │              └──────────┬──────────────────────────┘
                   │              ┌──────────▼──────────────────────────┐
                   │              │  5. LLM 流式生成 + SSE 推送           │
                   │              │  · Prompt 动态注入（意图→领域 Prompt）  │
                   │              │  · [REFERENCE] 引用溯源事件            │
                   │              │  · [CARD] 结构化卡片事件               │
                   │              └─────────────────────────────────────┘
                   └─────────────────────────────────────────────────────┘
```

**文档入库通用流程**：

```
文件上传 → 格式转换 → 智能切片 → 向量化嵌入 → 索引存储
         (PDF/Word/Excel)  (Splitter)   (Embedding)  (ES/PG/Milvus)
                                    ↓
                              事件驱动自动触发
                              + 分布式锁并发保护
                              + 定时任务补偿兜底
```

详细实现见 [agent.md](.claude/roles/agent.md) / [agent-rag.md](.claude/roles/agent-rag.md)。

---

## 三、角色索引（任务特征 → 自动加载 role.md）

> **核心原则**：AI 收到任务后，**必须**先看本表 → 自动加载对应 role.md → 写代码前再写。
>
> 一个任务可同时加载多个角色（数据流方向：设计 → 实现 → 数据 → 测试 → 部署 → 审查）。
>
> **🔥 国产模型约束**：所有 LLM 调用**必须用国产大模型**（主力 DeepSeek），**禁止 OpenAI / Anthropic / Cohere / Jina / OpenRouter**。详见 [agent.md §1.1](.claude/roles/agent.md) 与各角色「禁止事项」。

| 任务特征 | 加载角色 | 关键产出 |
|---------|---------|---------|
| 修改前端组件、聊天界面、SSE 消息解析、Pinia store | [frontend.md](.claude/roles/frontend.md) | Vue3 + 聊天 UI |
| FastAPI 路由 / Service / Pydantic Schema / 依赖注入 | [backend-fastapi.md](.claude/roles/backend-fastapi.md) | 异步 API 层 |
| Flask Blueprint / marshmallow / Celery 同步场景 | [backend-flask.md](.claude/roles/backend-flask.md) | 同步 API 层 |
| SQLAlchemy 模型 / Alembic 迁移 / PGVector 列 / seed | [dba.md](.claude/roles/dba.md) | 数据层 |
| Docker Compose / vLLM 部署 / 监控 / CI/CD | [devops.md](.claude/roles/devops.md) | 部署脚本 |
| PR Review / 代码审查 / 合并前检查 | [reviewer.md](.claude/roles/reviewer.md) | 审查清单 + 输出格式 |
| **接口集成测试**（testcontainers / 8 类用例 / 契约审查） | [integration-test.md](.claude/roles/integration-test.md) | 真实 DB 测试 |
| **RAG / Agent / Prompt / 向量库 / Memory / LLMOps**（最关键） | [agent.md](.claude/roles/agent.md) | 检索/生成/Memory/可观测 |
| RAG 检索增强 / VectorStore / 混合检索 / Re-ranking | [agent-rag.md](.claude/roles/agent-rag.md) | 检索方案 |
| Memory / Langfuse 可观测 / Guardrails / 失败回退 | [agent-llmops.md](.claude/roles/agent-llmops.md) | 可观测 + 兜底 |
| Token 预算 / 成本监控 / 评估流水线 / 延迟优化 | [agent-cost.md](.claude/roles/agent-cost.md) | 成本控制 |
| Prompt 模板 / Few-shot / RAG Prompt 优化 | [prompt-engineering.md](.claude/roles/prompt-engineering.md) | Prompt 设计 |
| MCP Server / Tool 设计 / Agent 集成 | [mcp.md](.claude/roles/mcp.md) | MCP 工具 |
| 创建 Skill / 触发优化 / Skill 编排 | [skill.md](.claude/roles/skill.md) | 知识包 |
| RAG 评测 / RAGAS / 指标 / CI 门禁 | [rag-evaluation.md](.claude/roles/rag-evaluation.md) | 评测流水线 |
| Multi-Agent 编排 / 任务分解 / 角色协作 | [multi-agent.md](.claude/roles/multi-agent.md) | 多 Agent 架构 |
| **安全规范**（API Key / 密码 / SQL 注入 / CORS / Prompt 注入 / JWT） | [security.md](.claude/roles/security.md) | 安全清单 |

> **手动指定优先级最高**：用户说"用 frontend 角色" / "用 agent 角色"时，绕过自动判断。

---

## 四、通用铁律（所有角色都遵守）

### 4.1 注释与命名

- **实体字段注释**：ORM `mapped_column()` 必须带 `comment=`；Pydantic / marshmallow Schema 字段必须带 `description`（缺一不可）
- **分步注释**：两步及以上逻辑的代码块，必须在对应代码行上方用 `# 第一步：xxx` / `# 第二步：xxx` 标注，**禁止写在行尾**
- 命名：变量/函数 `snake_case`、类 `PascalCase`、常量 `UPPER_CASE`（详细见对应 role.md）

### 4.2 设计模式与封装

| 模式 | 适用场景 | Python 实现 |
|------|---------|------------|
| 工厂 | 多种策略可切换（切片器 / 序列化器） | 函数返回实例 / dict 映射 |
| 策略 | 同一接口多种算法（检索 / 改写 / 降级） | `Protocol` / `ABC` + 字典注册 |
| 装饰器 | 横切关注点（日志 / 权限 / 缓存 / 限流） | Python 原生 `@decorator` |
| 观察者 | 事件驱动（入库→切片→向量化） | 回调 / `asyncio.Event` / 信号量 |
| 单例 | 全局共享资源（连接池 / LLM Client） | 模块级变量（Python 模块天然单例） |
| Repository | 封装数据访问，业务层不感知 SQL | 类封装 SQLAlchemy 查询 |
| 依赖注入 | 跨层解耦（API → Service → Repository） | FastAPI `Depends` |

- **工具类**：同一功能出现 ≥ 2 次必须提取到 `common/utils.py` 或 `app/utils/`，按职责拆文件，禁止在工具中 import 业务模块
- **Client 封装**：所有外部服务调用必须通过 `app/clients/` 下的 Client 类封装（统一初始化 / 异常处理 / 超时重试 / 日志）

### 4.3 角色文件使用规范

1. **命令式口吻，强约束**：使用「必须」「禁止」「不用」「强制」等词，不用「建议」「可以」
2. **❌/✅ 反例对照**：在关键设计/取舍点必须给出 ❌ 错误示例 + ✅ 正确示例
3. **代码即规范**：所有代码片段必须可直接复制使用（含完整 import / 版本号 / 配置项）
4. **版本固定**：技术栈表中的所有版本号必须精确（不能写"最新"）
5. **末尾必带「禁止事项」**：每个 role.md 末尾必须有「禁止事项」清单
6. **跨角色引用**：用 `[角色名](文件名.md#章节锚点)` 形式，形成「网状约束」

### 4.4 中间件与测试环境（强制）

**新增中间件时（按顺序执行）**：

1. 检查端口占用，启动前确认
2. 写 `docker-compose.yml`（含 image / ports / volumes / healthcheck）
3. `docker compose up -d {service_name}`
4. 健康检查
5. 更新 `.env.example` 的连接地址

**代码改动需要测试时**：

1. 检查测试依赖端口
2. 端口冲突判断：空闲直接启动 / 被占但正常直接测 / 被占且异常提示用户
3. 执行测试
4. 临时启动的服务，测试后询问用户是否关闭

详细 → [devops.md](.claude/roles/devops.md)。

### 4.5 禁止项

- ❌ 任何 LLM 调用使用 OpenAI / Anthropic / Cohere / Jina / OpenRouter
- ❌ 引入 `openai` / `anthropic` SDK 作为生产依赖
- ❌ 测试中调真实 LLM API（耗时 + 费用 + 不确定性），必须 mock 且 mock 为 `deepseek-chat`
- ❌ 降级链兜底不是国产模型
- ❌ 密码明文 / MD5 / SHA1（用 bcrypt / argon2）
- ❌ API Key / JWT Secret 硬编码或提交 git
- ❌ `print()` 调试（用 `logger.debug`）
- ❌ 新增代码引入 `# type: ignore`（除非明确理由并注释）
- ❌ 越权测试用同账号 + 不同权限角色（必须用第二个独立用户）
- ❌ 测试数据硬编码 `id=1` 等固定值（动态查询）
- ❌ 后端测试用 SQLite 替代 PostgreSQL（语法差异漏测生产 bug）
- ❌ 跑测试不启动 Docker（testcontainers 失败）
- ❌ Prompt 变更不上 Prompt Registry / 不跑 eval
- ❌ LLM 调用不设 `max_tokens`（成本失控）

---

## 五、命令速查

```bash
# 项目初始化
uv init my-project && cd my-project

# 依赖管理
uv add fastapi sqlalchemy
uv add --dev pytest ruff mypy
# Windows cmd/PowerShell 也可用：uv sync

# 开发运行
uv run uvicorn app.main:app --reload --port 8000
uv run flask --app app run --debug

# 测试
uv run pytest
uv run pytest tests/api/                # 接口集成测试
uv run pytest -k "test_login"           # 关键字过滤
uv run pytest --cov=app                 # 覆盖率

# 代码质量
uv run ruff check .
uv run ruff format .
uv run mypy app/

# 数据库迁移
uv run alembic revision --autogenerate -m "描述"
uv run alembic upgrade head
uv run alembic downgrade -1
```

> **平台说明**：`uv run` 在 Windows / MSYS / WSL / Linux / macOS 下通用；Unix shell 脚本（`#!/usr/bin/env bash`）仅限 Linux/macOS/WSL，Windows 本地需 WSL 或 Git Bash。

---

## 六、代码改动后验证流程（强制）

```
1. 改代码
2. uv run ruff check . && uv run mypy app/        # lint + 类型
3. 涉及接口 → uv run pytest tests/api/            # 真实 DB 集成测试
4. 涉及 LLM → uv run pytest tests/eval/           # RAGAS 离线评估
5. 需要中间件（PG/Redis/MinIO/vLLM）→ docker compose up -d
6. 启动顺序：中间件 → 后端 → 前端 → 测试接口
```

详细 Testcontainers 配置、8 类用例模板、契约审查、CI 接入 → [integration-test.md](.claude/roles/integration-test.md)。

---

## 七、Prompt 管理（强制）

- Prompt **必须**以文件存储（`.txt` / `.jinja2` / `.yaml`），**禁止硬编码在 Python 代码**
- 统一放 `prompts/` 或 `app/prompts/`，按业务域命名：`{domain}-{action}-prompt.txt`
- 用 Jinja2 模板语法注入变量：`{{ user_query }}`，禁止 f-string 拼接
- Prompt 文件纳入 Git 版本控制
- 重大 Prompt 变更需 A/B 测试验证效果后再上线
- 模板中必须有过滤占位符，用户输入先消毒再注入
- 禁止在 Prompt 中暴露系统架构、内部 API、数据库结构

详细 → [prompt-engineering.md](.claude/roles/prompt-engineering.md)。

---

## 八、错误处理与降级（强制）

**分层异常体系**（继承统一基类，**禁止**裸 `except Exception`）：

| 异常类型 | HTTP 状态码 | 场景 |
|----------|-----------|------|
| `BizException` | 400 | 业务校验失败 |
| `UnauthorizedException` | 401 | 未认证 |
| `NotFoundException` | 404 | 资源不存在 |
| `ExternalServiceException` | 502 | 第三方异常 |
| `LLMException` | 503 | LLM 调用失败 |

**LLM 重试**（tenacity，只重试可恢复错误，指数退避 1→60s，3 次）：

```python
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

@retry(
    retry=retry_if_exception_type((RateLimitError, TimeoutError, ConnectionError)),
    wait=wait_exponential(multiplier=1, min=2, max=60),
    stop=stop_after_attempt(3),
)
async def call_llm(prompt: str) -> str: ...
```

**降级策略**（关键链路必须有）：

- LLM 主力不可用 → 切备用国产模型（`deepseek-chat` → `qwen-plus`）
- 向量库不可用 → 降级全文检索
- 第三方 API 不可用 → 返回缓存或默认值

---

## 九、参考资源索引

> 各角色文件中不再单独列出参考来源，统一在此管理。

**AI/LLM 权威来源**：

- OpenAI Prompt Engineering: https://platform.openai.com/docs/guides/prompt-engineering
- Anthropic Prompt Engineering: https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/
- Google Prompting Strategies: https://ai.google.dev/gemini-api/docs/prompting-strategies
- LangChain: https://python.langchain.com/
- LangGraph: https://langchain-ai.github.io/langgraph/
- RAGAS: https://docs.ragas.io/
- MCP Specification: https://spec.modelcontextprotocol.io/
- DeepSeek API: https://platform.deepseek.com/api-docs
- 通义千问: https://help.aliyun.com/zh/model-studio/

**工具**：

- LangChain Prompt Hub: https://smith.langchain.com/hub
- Context7: MCP 工具 `mcp__context7__resolve-library-id` + `mcp__context7__query-docs`（查询最新官方文档）
- Langfuse: https://langfuse.com/
- MCP Inspector: 调试 MCP Server 可视化工具
