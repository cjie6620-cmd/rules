# rules

Claude Code 开发规范集合，按技术栈拆分为独立的 CLAUDE.md 规则包，直接拷贝到项目根目录即可使用。

## 目录结构

```
rules/
├── java-rules/      # Java 全栈规范
│   ├── CLAUDE.md
│   └── .claude/roles/
│       ├── backend-*.md   # 后端分层规范（单体/微服务/缓存/MQ/ES 等）
│       ├── frontend.md    # Vue3 前端规范
│       ├── dba.md         # 数据库设计规范
│       ├── devops.md      # CI/CD 与部署规范
│       └── reviewer.md    # Code Review 规范
│
└── python-rules/    # Python LLM 应用规范
    ├── CLAUDE.md
    └── .claude/roles/
        ├── agent*.md          # Agent / Multi-Agent / LLMops
        ├── backend-*.md       # FastAPI / Flask 后端规范
        ├── rag-*.md           # RAG 与检索评估规范
        ├── mcp.md             # MCP Server 开发规范
        ├── prompt-engineering.md
        ├── security.md
        ├── devops.md
        └── reviewer.md
```

## java-rules

适用于 **Spring Boot + Vue3 前后端分离**项目。

| 方向 | 技术栈 |
|------|--------|
| 前端 | Vue3 + Vite + Ant Design Vue |
| 后端 | Spring Boot 2.7 + Java 8 + MyBatis-Plus |
| 数据库 | MySQL 8.0 + Redis + Elasticsearch 7.x |
| 微服务 | Spring Cloud Alibaba（Nacos + Sentinel） |
| 中间件 | XXL-Job + RocketMQ |

涵盖：分层架构、单体/微服务切换、AOP、异步、缓存、ES、文件处理、数据库优化、前端组件、DevOps 等 18 个角色规范。

## python-rules

适用于 **FastAPI / Flask + RAG + Agent** 项目。

| 方向 | 说明 |
|------|------|
| RAG 全流程 | 意图识别 → 查询改写 → 路由 → 检索 → 重排 → 生成 |
| Agent | 单 Agent / Multi-Agent / 成本控制 / LLMops |
| 后端 | FastAPI / Flask 分层规范 |
| 工程化 | MCP Server、Prompt Engineering、安全、CI/CD |

涵盖：RAG 管道、Agent 编排、LLM 成本优化、检索评估、安全加固等 15 个角色规范。

## 使用方式

```bash
# 拷贝到你的项目根目录
cp -r java-rules/CLAUDE.md your-project/CLAUDE.md
cp -r java-rules/.claude your-project/.claude

# 或 Python 项目
cp -r python-rules/CLAUDE.md your-project/CLAUDE.md
cp -r python-rules/.claude your-project/.claude
```

Claude Code 会自动读取项目根目录的 `CLAUDE.md` 和 `.claude/roles/` 下的角色文件，按规范约束生成代码。

## 自定义

`CLAUDE.md` 中的 `xxx` 均为占位符，使用时替换为实际业务模块名即可。
