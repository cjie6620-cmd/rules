# 接口集成测试规范（API Integration Test，真实 DB 交互）

> 适用：Python 3.12+ + FastAPI 0.136+ + SQLAlchemy 2.0 async + PostgreSQL 16。
>
> 核心约束：**必须真实连库**（testcontainers 启 PG 16 + pgvector），禁止 Mock Repository。
>
> 角色定位：本文件只规定**接口集成测试的完整标准**。审查清单见 [reviewer.md](reviewer.md)，代码改动后验证流程见 [CLAUDE.md](../CLAUDE.md)。


## 1. 测试分层定位

| 层级 | 工具 | 范围 | 启动开销 | 数量 |
|------|------|------|---------|------|
| 单元测试 | pytest + pytest-mock | Service / Util 纯逻辑 | < 1s/类 | 多 |
| **接口集成测试（本文件）** | **pytest + httpx AsyncClient + testcontainers[postgresql]** | **API → Service → Repository → PG** | **30~60s 首次，复用 < 5s** | **每个 router ≥ 8 个** |
| E2E | Playwright | 浏览器链路 | > 1min | 少 |

> 接口集成测试 = 启动 FastAPI 容器（in-memory）+ 真实 HTTP 请求 + 真实 PG CRUD，但不启动浏览器。


## 2. 技术栈与版本（pyproject.toml 必加依赖）

```toml
[project.optional-dependencies]
test = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-cov>=5.0",
    "pytest-allure-adaptor>=2.13",         # 或 allure-pytest
    "httpx>=0.27",                          # AsyncClient
    "testcontainers[postgresql]>=4.0",      # 真实 PG 容器
    "asgi-lifespan>=2.0",                   # 启停 FastAPI lifespan
]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests/api"]
addopts = "-v --strict-markers"
markers = [
    "auth: 鉴权相关用例",
    "permission: 越权 / 权限相关",
    "boundary: 边界值",
]
```

**前置条件**：

- 本地 Docker Desktop 已启动（testcontainers 调用 Docker API 启动容器）
- `~/.testcontainers.properties` 配置复用（见 §3）


## 3. 项目目录结构

```
tests/
├── api/                                 # 接口集成测试根目录
│   ├── conftest.py                      # 全局 fixture（容器 + app + client）
│   ├── test_login.py                    # 登录端点
│   ├── test_user.py                     # 用户端点
│   └── test_role.py                     # 角色端点
└── resources/
    └── init_schema.sql                  # 建表 + admin 种子
```


## 4. conftest.py（完整可运行代码）

**第一步：Testcontainers 复用配置**（`~/.testcontainers.properties`）

```properties
testcontainers.reuse.enable=true
testcontainers.checks.disable=false
```

**第二步：conftest.py**

```python
"""
接口集成测试全局 fixture。

职责：
1. 启动 PostgreSQL 16 容器（含 pgvector 扩展）
2. 注入 JDBC URL 到 FastAPI Settings
3. 建表 + 种子数据
4. 提供 AsyncClient + 鉴权 token 工厂
"""
from __future__ import annotations

import asyncio
from typing import AsyncGenerator

import pytest
import pytest_asyncio
from httpx import ASGITransport, AsyncClient
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from testcontainers.postgres import PostgresContainer
from app.main import app
from app.core.config import settings
from app.core.security import create_access_token
from app.models.base import Base
from app.models.user import User  # noqa: F401  触发 metadata 注册
from app.models.role import Role  # noqa: F401


@pytest.fixture(scope="session")
def event_loop():
    """覆盖 pytest-asyncio 默认 loop，整个 session 共用一个。"""
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()


@pytest.fixture(scope="session")
def postgres_container() -> PostgresContainer:
    """PostgreSQL 16-alpine 容器（与生产版本严格一致，禁止 latest）。"""
    container = (
        PostgresContainer("postgres:16-alpine")
        .with_database("example_test")
        .with_username("test")
        .with_password("test123")
        .with_env("POSTGRES_INITDB_ARGS", "--data-checksums")
    )
    container.with_reuse(True)  # 需 ~/.testcontainers.properties
    container.start()

    # 启用 pgvector 扩展（Python LLM 项目常用）
    engine = create_async_engine(container.get_connection_url().replace("postgresql", "postgresql+asyncpg"))
    async with engine.begin() as conn:
        await conn.execute(text("CREATE EXTENSION IF NOT EXISTS vector"))
    asyncio.get_event_loop().run_until_complete(engine.dispose())

    return container


@pytest_asyncio.fixture(scope="session")
async def engine(postgres_container):
    """异步 SQLAlchemy engine，注入容器 JDBC URL。"""
    db_url = postgres_container.get_connection_url().replace("postgresql", "postgresql+asyncpg")
    eng = create_async_engine(db_url, echo=False, pool_size=10)
    async with eng.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    # 同步注入到全局 settings
    settings.DATABASE_URL = db_url
    yield eng
    await eng.dispose()


@pytest_asyncio.fixture
async def db_session(engine) -> AsyncGenerator[AsyncSession, None]:
    """每个测试一个独立 session，自动 rollback 隔离。"""
    session_maker = async_sessionmaker(engine, expire_on_commit=False)
    async with session_maker() as session:
        yield session
        await session.rollback()


@pytest_asyncio.fixture(autouse=True)
async def _clean_tables(db_session: AsyncSession):
    """每个测试前 TRUNCATE 业务表。"""
    yield  # 测试先跑
    await db_session.execute(
        text("TRUNCATE TABLE sys_user_role, sys_user, sys_role RESTART IDENTITY CASCADE")
    )
    await db_session.commit()


@pytest_asyncio.fixture
async def seeded_db(db_session: AsyncSession):
    """插入 admin / user01 基础账号。"""
    # 第一步：创建基础角色
    admin_role = Role(code="ADMIN", name="管理员")
    user_role = Role(code="USER", name="普通用户")
    db_session.add_all([admin_role, user_role])

    # 第二步：创建基础用户
    admin = User(username="admin", password_hash=hash_password("admin123"), role=admin_role)
    user01 = User(username="user01", password_hash=hash_password("user123"), role=user_role)
    db_session.add_all([admin, user01])
    await db_session.commit()
    return {"admin": admin, "user01": user01, "admin_role": admin_role, "user_role": user_role}


@pytest_asyncio.fixture
async def client(engine) -> AsyncGenerator[AsyncClient, None]:
    """FastAPI AsyncClient（lifespan 自动管理）。"""
    from app.dependencies import get_db  # 注入 DB session 依赖

    # 第一步：覆盖 DB 依赖
    async def _override_db():
        session_maker = async_sessionmaker(engine, expire_on_commit=False)
        async with session_maker() as session:
            yield session

    app.dependency_overrides[get_db] = _override_db

    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://testserver") as ac:
        yield ac

    app.dependency_overrides.clear()


@pytest_asyncio.fixture
async def admin_token(client: AsyncClient, seeded_db) -> str:
    """获取 admin Bearer Token。"""
    res = await client.post(
        "/api/v1/auth/login",
        json={"username": "admin", "password": "admin123", "captcha": "0000", "captcha_id": "test"},
    )
    assert res.status_code == 200
    return f"Bearer {res.json()['data']['token']}"


@pytest_asyncio.fixture
async def user_token(client: AsyncClient, seeded_db) -> str:
    """获取 user01 Bearer Token。"""
    res = await client.post(
        "/api/v1/auth/login",
        json={"username": "user01", "password": "user123", "captcha": "0000", "captcha_id": "test"},
    )
    assert res.status_code == 200
    return f"Bearer {res.json()['data']['token']}"
```


## 5. 用例设计矩阵（每个端点至少覆盖 8 类）

| # | 用例类型 | 必含 | 验证目标 |
|---|---------|------|---------|
| 1 | **正常路径** | ✓ | 200 + 业务码 0/200 + 数据正确 |
| 2 | **参数校验** | ✓ | 422（Pydantic 校验） + 业务码 400 + msg 明确 |
| 3 | **未鉴权** | ✓ | 401 + 业务码 401 |
| 4 | **越权** | ✓ | 403 + 业务码 403（**不返 404 避免泄露**） |
| 5 | **资源不存在** | ✓ | 404 + 业务码 404 |
| 6 | **业务冲突** | ✓ | 409 / 423 / 422 视业务而定 |
| 7 | **边界值** | ✓ | min / max / 空串 / None / 超长 |
| 8 | **幂等性** | 视情况 | 重复请求应得相同结果（PUT / DELETE） |


## 6. 三端点完整示例

### 6.1 test_login.py（登录端点，覆盖 5 次密码错误锁定）

```python
"""
登录接口集成测试。
"""
from __future__ import annotations

import pytest
from httpx import AsyncClient


pytestmark = pytest.mark.asyncio


async def test_login_success(client: AsyncClient, seeded_db):
    """正常登录：admin 账号 + 正确密码 → 200 + 返回 token。"""
    res = await client.post(
        "/api/v1/auth/login",
        json={"username": "admin", "password": "admin123", "captcha": "0000", "captcha_id": "test"},
    )
    assert res.status_code == 200
    body = res.json()
    assert body["code"] == 0  # FastAPI 业务码
    assert body["data"]["token"]
    assert body["data"]["expires_in"] > 0


async def test_login_locked_after_5_failures(client: AsyncClient, seeded_db):
    """密码错误 5 次 → 第 6 次返回 423 Locked。"""
    payload = {"username": "admin", "password": "wrong_pwd", "captcha": "0000", "captcha_id": "test"}

    # 连续 5 次错误密码
    for _ in range(5):
        res = await client.post("/api/v1/auth/login", json=payload)
        assert res.status_code == 200
        assert res.json()["code"] == 401

    # 第 6 次触发锁定
    res = await client.post("/api/v1/auth/login", json=payload)
    assert res.status_code == 423
    body = res.json()
    assert body["code"] == 423
    assert "锁定" in body["message"]


async def test_login_missing_username(client: AsyncClient, seeded_db):
    """参数缺失：username 为空 → 422。"""
    res = await client.post(
        "/api/v1/auth/login",
        json={"password": "admin123", "captcha": "0000"},
    )
    assert res.status_code == 422  # Pydantic 校验
    body = res.json()
    assert "username" in str(body)


async def test_login_user_not_exist(client: AsyncClient, seeded_db):
    """账号不存在 → 401 + 模糊提示（不暴露账号是否存在）。"""
    res = await client.post(
        "/api/v1/auth/login",
        json={"username": "ghost_user_xxx", "password": "any", "captcha": "0000", "captcha_id": "test"},
    )
    assert res.status_code == 200
    body = res.json()
    assert body["code"] == 401
    assert ("账号" in body["message"]) or ("密码" in body["message"])


async def test_login_wrong_captcha(client: AsyncClient, seeded_db):
    """Captcha 错误 → 400。"""
    res = await client.post(
        "/api/v1/auth/login",
        json={"username": "admin", "password": "admin123", "captcha": "9999", "captcha_id": "test"},
    )
    assert res.status_code == 400
    assert res.json()["code"] == 400
```

### 6.2 test_user.py（用户端点，覆盖 CRUD + 越权）

```python
"""
用户管理接口集成测试。
"""
from __future__ import annotations

import time

import pytest
from httpx import AsyncClient


pytestmark = pytest.mark.asyncio


async def test_list_users_as_admin(client: AsyncClient, admin_token: str):
    """分页查询用户列表：ADMIN → 200 + total/list 结构。"""
    res = await client.get(
        "/api/v1/users",
        params={"page": 1, "page_size": 10},
        headers={"Authorization": admin_token},
    )
    assert res.status_code == 200
    body = res.json()
    assert body["code"] == 0
    assert body["data"]["total"] >= 0
    assert isinstance(body["data"]["list"], list)


async def test_list_users_as_user_forbidden(client: AsyncClient, user_token: str):
    """分页查询：USER 角色越权 → 403。"""
    res = await client.get(
        "/api/v1/users",
        params={"page": 1, "page_size": 10},
        headers={"Authorization": user_token},
    )
    assert res.status_code == 200
    assert res.json()["code"] == 403


async def test_list_users_unauthorized(client: AsyncClient):
    """未携带 Token → 401。"""
    res = await client.get("/api/v1/users")
    assert res.status_code == 200
    assert res.json()["code"] == 401


async def test_create_user_duplicate(client: AsyncClient, admin_token: str):
    """创建用户：用户名重复 → 业务码 409。"""
    res = await client.post(
        "/api/v1/users",
        headers={"Authorization": admin_token},
        json={
            "username": "admin",  # 已存在
            "password": "Test123!",
            "nickname": "测试",
            "email": "test@dup.com",
            "role_ids": [2],
        },
    )
    assert res.status_code == 200
    body = res.json()
    assert body["code"] == 409
    assert "用户名已存在" in body["message"]


async def test_create_user_invalid_email(client: AsyncClient, admin_token: str):
    """创建用户：邮箱格式错误 → 422（Pydantic EmailStr 校验）。"""
    res = await client.post(
        "/api/v1/users",
        headers={"Authorization": admin_token},
        json={
            "username": f"new_{int(time.time() * 1000)}",
            "password": "Test123!",
            "nickname": "测试",
            "email": "not-an-email",
            "role_ids": [2],
        },
    )
    assert res.status_code == 422
    body = res.json()
    assert "email" in str(body).lower()


async def test_create_user_weak_password(client: AsyncClient, admin_token: str):
    """创建用户：弱密码（< 8 位）→ 400。"""
    res = await client.post(
        "/api/v1/users",
        headers={"Authorization": admin_token},
        json={
            "username": f"new_{int(time.time() * 1000)}",
            "password": "123",
            "nickname": "测试",
            "email": "ok@example.com",
            "role_ids": [2],
        },
    )
    assert res.status_code == 400
    assert res.json()["code"] == 400


async def test_update_my_profile(client: AsyncClient, user_token: str):
    """更新用户：用户改自己昵称 → 200。"""
    res = await client.put(
        "/api/v1/users/me",
        headers={"Authorization": user_token},
        json={"nickname": "新昵称"},
    )
    assert res.status_code == 200
    body = res.json()
    assert body["code"] == 0
    assert body["data"]["nickname"] == "新昵称"


async def test_delete_user_forbidden(client: AsyncClient, user_token: str):
    """删除用户：USER 越权 → 403。"""
    res = await client.delete(
        "/api/v1/users/1",
        headers={"Authorization": user_token},
    )
    assert res.status_code == 200
    assert res.json()["code"] == 403


async def test_delete_user_not_found(client: AsyncClient, admin_token: str):
    """删除用户：ADMIN 删除不存在用户 → 404。"""
    res = await client.delete(
        "/api/v1/users/999999",
        headers={"Authorization": admin_token},
    )
    assert res.status_code == 200
    assert res.json()["code"] == 404
```

### 6.3 test_role.py（角色端点，覆盖绑定 + 越权）

```python
"""
角色管理接口集成测试。
"""
from __future__ import annotations

import pytest
from httpx import AsyncClient


pytestmark = pytest.mark.asyncio


async def test_list_roles_as_user(client: AsyncClient, user_token: str):
    """查询角色列表：USER 可读 → 200。"""
    res = await client.get(
        "/api/v1/roles",
        headers={"Authorization": user_token},
    )
    assert res.status_code == 200
    body = res.json()
    assert body["code"] == 0
    assert isinstance(body["data"], list)
    assert len(body["data"]) > 0


async def test_create_role_forbidden(client: AsyncClient, user_token: str):
    """创建角色：USER 越权 → 403。"""
    res = await client.post(
        "/api/v1/roles",
        headers={"Authorization": user_token},
        json={"code": "TESTER", "name": "测试员", "permission_ids": [1]},
    )
    assert res.status_code == 200
    assert res.json()["code"] == 403


async def test_create_role_duplicate_code(client: AsyncClient, admin_token: str):
    """创建角色：ADMIN + 重复 code → 409。"""
    res = await client.post(
        "/api/v1/roles",
        headers={"Authorization": admin_token},
        json={"code": "ADMIN", "name": "管理员", "permission_ids": []},
    )
    assert res.status_code == 200
    assert res.json()["code"] == 409


async def test_bind_role_self_forbidden(client: AsyncClient, user_token: str):
    """绑定角色：USER 改自己 → 403（仅 ADMIN 可改）。"""
    res = await client.put(
        "/api/v1/users/me/roles",
        headers={"Authorization": user_token},
        json={"role_ids": [2]},
    )
    assert res.status_code == 200
    assert res.json()["code"] == 403


async def test_bind_role_not_found(client: AsyncClient, admin_token: str):
    """绑定角色：ADMIN 绑定不存在角色 → 404。"""
    res = await client.put(
        "/api/v1/users/2/roles",
        headers={"Authorization": admin_token},
        json={"role_ids": [99999]},
    )
    assert res.status_code == 200
    assert res.json()["code"] == 404


async def test_delete_role_in_use(client: AsyncClient, admin_token: str):
    """删除角色：被引用时（admin）→ 409 + 业务提示。"""
    res = await client.delete(
        "/api/v1/roles/1",
        headers={"Authorization": admin_token},
    )
    assert res.status_code == 200
    body = res.json()
    assert body["code"] == 409
    assert "正在被" in body["message"]


async def test_delete_role_as_admin_forbidden(client: AsyncClient, admin_token: str):
    """删除角色：ADMIN 越权删 → 403（仅 SUPER_ADMIN 可删）。"""
    res = await client.delete(
        "/api/v1/roles/2",
        headers={"Authorization": admin_token},
    )
    assert res.status_code == 200
    assert res.json()["code"] == 403
```


## 7. 数据准备与清理

**策略 A：每测试自动 TRUNCATE（推荐，隔离最强）**

```python
# conftest.py 中的 _clean_tables fixture 已自动处理
@pytest_asyncio.fixture(autouse=True)
async def _clean_tables(db_session: AsyncSession):
    yield
    await db_session.execute(
        text("TRUNCATE TABLE sys_user_role, sys_user, sys_role RESTART IDENTITY CASCADE")
    )
    await db_session.commit()
```

**策略 B：每测试函数 SAVEPOINT + 回滚（速度最快，但跨事务逻辑测不到）**

```python
@pytest_asyncio.fixture
async def db_session(engine):
    async with engine.connect() as conn:
        trans = await conn.begin()
        async_session = AsyncSession(bind=conn, expire_on_commit=False)
        try:
            yield async_session
        finally:
            await trans.rollback()
            await async_session.close()
```

> **推荐组合**：策略 A 兜底（兼容性最强）+ 关键事务提交逻辑用 SAVEPOINT 单独测。


## 8. 执行命令与报告输出

```bash
# 跑全部接口测试
uv run pytest tests/api/

# 跑单个端点
uv run pytest tests/api/test_login.py

# 跑单个用例
uv run pytest tests/api/test_login.py::test_login_locked_after_5_failures

# 跑鉴权相关（按 marker 过滤）
uv run pytest tests/api/ -m "auth or permission"

# 覆盖率
uv run pytest tests/api/ --cov=app --cov-report=html

# Allure 报告
uv run pytest tests/api/ --alluredir=./allure-results
uv run allure serve allure-results
```

**测试输出标准**：

```
============= 28 passed in 12.34s =============
```


## 9. CI 接入（GitHub Actions 片段）

```yaml
name: api-integration-test

on: [pull_request]

jobs:
  api-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v3

      - name: Set up Python
        run: uv python install 3.12

      - name: Install dependencies
        run: uv sync --all-extras

      - name: Run API integration tests
        run: uv run pytest tests/api/ -v --tb=short

      - name: Upload Allure results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: allure-results
          path: allure-results/
```


## 12. 前后端联调代码审查规范

> 接口集成测试验证后端 API 正确性，本节从**代码审查角度**验证前后端联调是否完整、数据流是否闭环。

### 12.1 组件-接口绑定审查清单（每个业务页面逐项核查）

**原则：前端每个按钮/下拉框/搜索框，必须能在代码中追溯到对应的 API 调用，且数据展示链路完整。**

| 组件类型 | 审查项 | 验证方式（代码层面） |
|----------|--------|---------------------|
| **搜索/查询按钮** | 是否调用正确的列表 API，query 参数名/类型与后端 Pydantic Schema 一致 | 搜索组件 `@click` → 找到 `api/xxx.ts` 函数 → 对比后端 `router` 参数 |
| **新增按钮** | 点击后是否打开表单弹窗，弹窗表单字段是否与后端 `BaseModel` 一一对应 | 按钮事件 → 弹窗组件 → 表单 `v-model` 字段 ↔ Schema 字段 |
| **编辑按钮** | 是否先调用详情 API 回显数据，表单字段值是否正确绑定 | `@click` → `getDetail(id)` → 表单 `v-model` 字段 ↔ 响应 data 字段 |
| **删除按钮** | 是否传递正确的 ID，是否有二次确认弹窗 | `@click` → `deleteById(row.id)` → `Modal.confirm` |
| **下拉框（Select）** | options 数据是否从 API 获取，`@change` 是否触发对应查询 | `options` 来源 → 哪个 API → `@change` 事件 → 列表刷新逻辑 |
| **分页组件** | `current`/`pageSize` 变化是否调用列表 API，参数名与后端一致 | `@change` 事件 → `fetchList({ page, page_size })` ↔ 后端 Query 参数 |
| **导出按钮** | 是否传递当前筛选条件到导出 API | `@click` → `exportXxx({ ...queryForm })` ↔ 后端导出接口参数 |
| **批量操作** | 勾选的 ID 集合是否正确传递，后端是否支持批量接口 | `selectedRowKeys` → `batchXxx(ids)` ↔ 后端 `ids: list[int]` |
| **SSE 流式对话** | 发送按钮调用 SSE 接口，消息列表是否正确累积 token | `sendMessage()` → `fetchEventSource()` → chatStore `appendToken()` |

### 12.2 数据流闭环审查（CRUD 全链路）

**每个业务实体必须验证以下 5 步在代码中全部存在：**

```
① 列表展示（搜索+分页）→ ② 新增 → ③ 编辑（回显+保存）→ ④ 删除 → ⑤ 列表刷新
```

| 步骤 | 代码审查点 | 典型缺失 |
|------|-----------|---------|
| ① 列表展示 | `onMounted` 中是否调用 `fetchList()`，搜索按钮 `@click` 是否调用 `fetchList()` | 新增后返回列表未刷新 |
| ② 新增 | 提交后是否 `emit('success')` 或调用 `fetchList()` 刷新父组件列表 | 提交成功后弹窗关闭但列表未更新 |
| ③ 编辑回显 | 打开弹窗时是否调用详情 API 获取数据，`v-model` 字段名与后端响应一致 | 编辑表单字段为空（未回显） |
| ③ 编辑保存 | 提交后是否用同一个 API（或 PUT/PATCH），参数名与后端一致 | 保存按钮调用了新增接口而非编辑接口 |
| ④ 删除 | 是否有 `Modal.confirm` 二次确认，传 `row.id` 而非硬编码 | 删除后列表未刷新，或传了错误 ID |
| ⑤ 刷新 | 增/删/改 成功后是否调用 `fetchList()` 或 `emit('refresh')` | 操作成功但列表不更新 |

### 12.3 异常处理与交互审查

| 场景 | 审查点 | 要求 |
|------|--------|------|
| **接口失败** | API 调用是否有 `catch` + toast 提示 | 禁止无错误处理的 API 调用 |
| **表单校验** | 提交前是否有前端校验（`rules`/`required`），校验规则与后端 Pydantic Schema 一致 | 前端 `required` ↔ 后端 `Field(...)`；前端 `type: 'email'` ↔ 后端 `EmailStr` |
| **空数据** | 列表为空时是否有占位提示（"暂无数据"） | 禁止空白页面 |
| **loading 状态** | 请求中是否显示 loading（骨架屏/spin），防止重复提交 | 按钮 `loading` 属性在请求期间为 `true` |
| **权限控制** | 按钮/菜单是否按角色显示隐藏，与后端 `Depends(require_role)` 一致 | `v-if="hasPermission('xxx:edit')"` ↔ 后端 `Depends(require_permission("xxx:edit"))` |

### 12.4 禁止事项

| 禁止 | 原因 |
|------|------|
| ❌ 前端组件调用了后端不存在的接口 | 联调 404，代码审查可发现 |
| ❌ 后端接口无前端调用（死接口） | 资源浪费，应清理 |
| ❌ 前端字段名与后端 Pydantic Schema 不一致（驼峰/下划线混用） | 序列化失败或字段丢失 |
| ❌ 编辑表单未调用详情 API 回显 | 用户看到空白表单 |
| ❌ 删除操作无二次确认 | 误删风险 |
| ❌ 操作成功后列表未刷新 | 用户以为操作失败 |
| ❌ 接口失败无 toast 提示 | 用户无感知 |
| ❌ 表单校验规则与后端不一致 | 前端校验通过但后端报错 |


## 13. 禁止事项

| 禁止 | 原因 |
|------|------|
| ❌ `AsyncMock` 跳过真实 DB | 违反「真实库交互」根本要求，测试失真 |
| ❌ 用 SQLite 内存数据库替代 PostgreSQL | 语法差异（`JSONB`、`ARRAY`、`pgvector`、日期函数）会导致生产 bug 漏测 |
| ❌ 测试中 `await asyncio.sleep(1)` 等待异步 | 用 `asyncio.wait_for` + `asyncio.Event` 替代 |
| ❌ 跨测试函数共享可变状态（`global` 字典 / 模块级变量） | 顺序依赖导致 CI 偶发失败 |
| ❌ 用生产库 URL 跑接口测试 | 误删数据，必须 testcontainers 隔离 |
| ❌ Postgres 镜像用 `latest` tag | 版本漂移导致「本地能跑 CI 挂」 |
| ❌ 测试函数不写中文 docstring | Allure 报告显示 `test_xxx` 无法定位失败 |
| ❌ 越权测试用同账号 + 不同权限角色 | 必须用**第二个独立用户**的 token |
| ❌ 测试数据硬编码 `id=1` 等固定值 | 用 `select(User).where(User.username == "admin")` 动态获取 |
| ❌ 不重置 Redis 失败计数就连续跑锁定测试 | 锁定状态污染后续用例 |
| ❌ 跑真实 LLM API（`await deepseek_chat(...)`） | 测试不可重现 + 烧钱 + 违反「禁止国外 API」 |
| ❌ `from openai import OpenAI` / `import anthropic` | 违反 [CLAUDE.md](../../CLAUDE.md) 国产模型铁律 |


## 11. 前后端接口契约一致性审查（API Contract Review，补充 17 项）

> 本节是 §6 §6.6 中"必查 6 项"的完整版。**前后端联调前**、**合并前**必须按本章逐项核对。
> 适用：[frontend.md](frontend.md) + [backend-fastapi.md](backend-fastapi.md) / [backend-flask.md](backend-flask.md) 同时变更的场景。

### 11.1 接口清单收集（必做，审查第一步）

**自动化收集**：

```bash
# 1. 收集后端 FastAPI / Flask 接口清单
grep -rE "@(router|app|blp)\.(get|post|put|delete|patch)\(" app \
  | sed -E 's/.*@(router|app|blp)\.(get|post|put|delete|patch)\(["'"'"']([^'"'"'"]+).*/\U\2\E \3/' \
  | sort -u > /tmp/backend-apis.txt

# 2. 收集前端 ofetch / fetch 调用清单
grep -rE "\$fetch\(.*method:|ofetch\(|fetch\(" frontend/src \
  | sed -E 's/.*[\$]?fetch\(["'"'"'`]*([^"'"'"'`,)]+).*/GET \1/' \
  | sort -u > /tmp/frontend-apis.txt

# 3. 取差集
diff /tmp/backend-apis.txt /tmp/frontend-apis.txt
```

**手工补充**：

- [ ] OpenAPI Schema 导出：`backend/openapi.json` 与前端 `src/api/*.ts` 是否一一对应
- [ ] Nginx / Kong 路由表（微服务）：`deploy/nginx.conf` 中 `location /api/`
- [ ] 前端路由 `src/router/*.ts` 中 `meta.api` 字段（若项目用此约定）

### 11.2 逐项核对清单（17 项，缺一即不通过）

**接口数量与基础信息**：

- [ ] **数量一致**：后端 `@router.get/post/...` 数量 == 前端 `api/*.ts` 中 export 函数数量（允许 ±0，**禁止**前端多调不存在的接口 / 后端存在无前端调用的「死接口」）
- [ ] **路径前缀一致**：`/api/v1/{module}` 与前端 `baseURL` 拼接后路径完全一致（注意 FastAPI 强制带 `/v1`）
- [ ] **HTTP 方法一致**：后端 `@router.get` 必须对应前端 `ofetch(url, { method: 'GET' })` 或 `$fetch`，错配 405
- [ ] **RESTful 语义正确**：`GET` 必须幂等且无 Body；`POST` 创建；`PUT` 全量更新；`PATCH` 局部更新；`DELETE` 删除（软删用 `PUT /{id}/disable`）

**请求参数**：

- [ ] **路径参数**：后端 `path_param: int` ↔ 前端 URL `/users/${id}`，类型与命名严格一致
- [ ] **Query 参数**：后端 `page: int = Query(1)` ↔ 前端 `query: { page: 1 }`，**禁止**前端用 `pageNum` 调 Python snake_case 后端
- [ ] **Body 参数**：Pydantic `BaseModel` 字段名（snake_case）、类型、必填项、嵌套结构与前端 `data: { ... }` 字段一一对应
- [ ] **日期格式**：后端 `datetime` 出参 ISO 8601（`2024-01-15T10:30:00+08:00`）↔ 前端 `dayjs` 解析格式一致
- [ ] **枚举值**：后端 `Literal["normal", "locked"]` ↔ 前端 `select options` 选项完全一致
- [ ] **分页参数**：后端 `page`(从 1 开始) + `page_size`(默认 10) ↔ 前端 `query: { page: 1, page_size: 10 }`

**响应结构**：

- [ ] **统一包装**：`R[T]` 字段 `code` / `message` / `data` / `trace_id` 必须三端对齐（前端拦截器 + 后端 `BizException` / `R.ok()`）
- [ ] **业务码**：成功 `code = 0`（FastAPI）/ `code = 200`（Flask），业务失败 `4xx` 段，系统异常 `5xx` 段
- [ ] **空值处理**：`None` 字段序列化为 `null`（Pydantic 默认），前后端字段必须一致
- [ ] **大数字**：Pydantic `conint(gt=2**53)` 需转 String，前端用 String 类型接收

**鉴权与跨域**：

- [ ] **Token 传递**：前端 `Authorization: Bearer ${token}` ↔ 后端 `Depends(get_current_user)` / `verify_jwt` 解析逻辑一致
- [ ] **CORS**：后端 `CORSMiddleware` `allow_origins` 允许前端 dev/prod 域名；预检 `OPTIONS` 不被中间件拦截
- [ ] **Cookie 模式**：若用 httpOnly cookie，前后端 `credentials: 'include'` / `SameSite` 配置必须匹配

### 11.3 不匹配清单输出格式（必填，附在 Review 评论）

```markdown
## 接口契约审查结果

### 缺失接口（后端存在但前端未调用）
- [严重] `GET /api/v1/admin/stats/overview` — 后端 admin_router.py:23 已实现，前端无对应 API 文件

### 多余接口（前端调用但后端不存在）
- [严重] `POST /api/v1/users/import` — 前端 user.ts:45 存在，后端无此端点（404）

### 路径不匹配
- [严重] 前端 `user.ts:30` → `GET /api/v1/user/list` ↔ 后端 `user_router.py:18` → `@router.get("/users/list")`（单复数 + 缺 v1）

### 参数不匹配
- [建议] 前端 `query: { pageNum: 1 }` ↔ 后端 `page: int`（命名风格冲突）

### 方法不匹配
- [严重] 前端 `ofetch(url, { method: 'DELETE' })` ↔ 后端 `@router.post("/users/{id}/delete")`（应改为 DELETE 或调整后端）

### 响应结构不匹配
- [严重] 前端期望 `data.items`，后端返回 `data.list`（字段名错配）
```

### 11.4 一致性对照表模板（PR 描述中强制附带）

| 序号 | 接口路径 | 方法 | 后端位置 | 前端位置 | 参数 | 响应 | 鉴权 | 状态 |
|------|---------|------|---------|---------|------|------|------|------|
| 1 | `/api/v1/auth/login` | POST | `auth_router.py:15` | `api/auth.ts:login()` | `{username, password, captcha}` | `{token, expires_in}` | 匿名 | ✅ |
| 2 | `/api/v1/users` | GET | `user_router.py:42` | `api/user.ts:list()` | `{page, page_size, keyword}` | `{total, list[]}` | ADMIN | ⚠️ 前端缺 keyword |
| 3 | `/api/v1/users/{id}` | DELETE | `user_router.py:88` | `api/user.ts:remove()` | path: `{id}` | `{id}` | SUPER_ADMIN | ✅ |

### 11.5 鉴权与角色矩阵（必须逐格打勾）

| 角色 | 可访问接口 |
|------|-----------|
| 匿名 | `POST /api/v1/auth/login`、`POST /api/v1/auth/captcha`、`GET /api/v1/auth/captcha-image` |
| USER | `GET /api/v1/users/me`、`PUT /api/v1/users/me/password`、所有 `GET /api/v1/roles`（仅查询） |
| ADMIN | 上述全部 + `/api/v1/users/*`（CRUD）、`/api/v1/roles/*`（CRUD） |
| SUPER_ADMIN | 上述全部 + `DELETE /api/v1/users/{id}`（软删）、`/api/v1/system/*`（系统配置） |

- [ ] 每个接口 `Depends(require_role("ADMIN"))` / `Depends(require_permission("user:create"))` 与上述矩阵 100% 对齐
- [ ] 前端路由 `meta.roles` 与后端 `require_role` 字符串完全一致（区分大小写）
- [ ] 越权访问（USER 调 `/api/v1/users/{id}` DELETE）必须返回 `code=403` 而非 `404`（避免信息泄露）

### 11.6 禁止事项

| 禁止 | 原因 |
|------|------|
| ❌ 前端 `fetch()` 硬编码完整 URL（含域名端口） | 切换环境需改源码，违反「配置外置」 |
| ❌ 后端 `@router.get("")` 不写路径（仅 prefix + 方法无路径） | 路径不明确，OpenAPI 文档失真 |
| ❌ 前后端用 `success` / `fail` 布尔字段而非 `code` 数字 | 无法区分业务错误类型 |
| ❌ 前端调用未在 `src/api/` 集中封装（在 `.vue` 里直接 `ofetch`） | 无法统一拦截器，token 注入 / 错误处理散落 |
| ❌ 路径含动词（`/api/v1/deleteUser/{id}`） | 不符合 RESTful，应为 `DELETE /api/v1/users/{id}` |
| ❌ 接口路径用驼峰（`/api/v1/userInfo`）或下划线（`/api/v1/user_info`） | 与 [CLAUDE.md](../../CLAUDE.md) snake_case 命名规范冲突 |
| ❌ 鉴权依赖缺失（`Depends(get_current_user)` 忘加） | 越权风险，CI 必须有静态扫描拦截 |
| ❌ 测试中 `import openai` / `from anthropic import ...` | 违反 [CLAUDE.md](../../CLAUDE.md) 「禁止国外 LLM API」铁律 |

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
