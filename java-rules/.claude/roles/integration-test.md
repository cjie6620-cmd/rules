# 接口集成测试规范（API Integration Test，真实 DB 交互）

> 适用：Java 8/17 + Spring Boot 2.7.18 + MyBatis-Plus 3.5.5 + MySQL 8.0.33。
>
> 核心约束：**必须真实连库**（Testcontainers 启 MySQL），禁止 Mock DAO / Mock Mapper。
>
> 角色定位：本文件只规定**接口集成测试的完整标准**。审查清单见 [reviewer.md](reviewer.md)，代码改动后验证流程见 [CLAUDE.md](../CLAUDE.md)。

---

## 1. 测试分层定位

| 层级 | 工具 | 范围 | 启动开销 | 数量 |
|------|------|------|---------|------|
| 单元测试 | JUnit 5 + Mockito | Service / Util 纯逻辑 | < 1s/类 | 多 |
| **接口集成测试（本文件）** | **Spring Boot Test + REST Assured + Testcontainers** | **Controller → Service → Mapper → MySQL** | **30~60s 首次，复用 < 5s** | **每个 Controller ≥ 8 个** |
| E2E | Selenium / Playwright | 浏览器链路 | > 1min | 少 |

> 接口集成测试 = 启动 Spring 容器 + 真实 HTTP 请求 + 真实 MySQL CRUD，但不启动浏览器。

---

## 2. 技术栈与版本（pom.xml 必加依赖）

```xml
<properties>
    <testcontainers.version>1.19.7</testcontainers.version>
    <rest-assured.version>5.3.2</rest-assured.version>
    <mysql-connector.version>8.0.33</mysql-connector.version>
    <junit-jupiter.version>5.10.1</junit-jupiter.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- REST Assured: 链式 HTTP 断言 -->
    <dependency>
        <groupId>io.rest-assured</groupId>
        <artifactId>rest-assured</artifactId>
        <version>${rest-assured.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Testcontainers: 真实 DB 容器 -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <version>${testcontainers.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>${testcontainers.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>mysql</artifactId>
        <version>${testcontainers.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Allure 报告（可选，强烈推荐） -->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-rest-assured</artifactId>
        <version>2.24.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

**前置条件**：

- 本地 Docker Desktop 已启动（Testcontainers 调用 Docker API 启动容器）
- `~/.testcontainers.properties` 配置复用（见 §3）

---

## 3. 项目目录结构

```
backend/src/test/java/
├── api/                                 # 接口集成测试根目录
│   ├── base/
│   │   ├── BaseApiTest.java             # 容器 + Spring 启动基类
│   │   └── AuthHelper.java              # 登录获取 Token 辅助
│   ├── login/
│   │   └── LoginApiTest.java            # 登录端点
│   ├── user/
│   │   └── UserApiTest.java             # 用户端点
│   └── role/
│       └── RoleApiTest.java             # 角色端点
└── resources/
    ├── application-test.yml             # 测试配置（指向容器 DB）
    └── sql/
        └── init-schema.sql              # 建表 + admin 种子
```

---

## 4. BaseTest 基类（完整可运行代码）

**第一步：Testcontainers 复用配置**（`~/.testcontainers.properties`）

```properties
testcontainers.reuse.enable=true
testcontainers.checks.disable=false
```

**第二步：BaseApiTest.java**

```java
package com.example.api.base;

import io.restassured.RestAssured;
import io.restassured.builder.RequestSpecBuilder;
import io.restassured.filter.log.LogDetail;
import io.restassured.http.ContentType;
import io.restassured.specification.RequestSpecification;
import org.junit.jupiter.api.BeforeEach;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.MySQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

/**
 * 接口集成测试基类。
 * 职责：启动 MySQL 8.0.33 容器 + Spring Boot 容器 + REST Assured 基础配置。
 * 所有 *ApiTest 必须继承本类，复用容器避免重复启动 30s+。
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@Testcontainers
public abstract class BaseApiTest {

    /**
     * MySQL 8.0.33 容器（与生产版本严格一致，禁止 latest）。
     * reuse=true 需 ~/.testcontainers.properties 配置 + 镜像带 "reuse" label。
     */
    @Container
    protected static final MySQLContainer<?> MYSQL =
        new MySQLContainer<>(DockerImageName.parse("mysql:8.0.33"))
            .withDatabaseName("example_test")
            .withUsername("test")
            .withPassword("test123")
            .withReuse(true)
            .withLabel("reuse", "true")
            .withInitScript("sql/init-schema.sql");

    /**
     * 动态注入容器 JDBC URL 到 Spring 配置文件。
     */
    @DynamicPropertySource
    static void registerProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", MYSQL::getJdbcUrl);
        registry.add("spring.datasource.username", MYSQL::getUsername);
        registry.add("spring.datasource.password", MYSQL::getPassword);
        registry.add("spring.datasource.driver-class-name", () -> "com.mysql.cj.jdbc.Driver");
    }

    @LocalServerPort
    protected int port;

    protected RequestSpecification requestSpec;

    @BeforeEach
    void setUp() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = port;
        RestAssured.basePath = "";
        RestAssured.enableLoggingOfRequestAndResponseIfValidationFails(LogDetail.ALL);

        this.requestSpec = new RequestSpecBuilder()
            .setContentType(ContentType.JSON)
            .setAccept(ContentType.JSON)
            .build();
    }
}
```

**第三步：application-test.yml**

```yaml
spring:
  datasource:
    url: jdbc:tc:mysql:8.0.33:///example_test?TC_REUSABLE=true
    driver-class-name: org.testcontainers.jdbc.ContainerDatabaseDriver
    username: test
    password: test123
  redis:
    host: localhost
    port: 6379
  sa-token:
    token-name: Authorization
    timeout: 2592000
logging:
  level:
    com.example: DEBUG
```

**第四步：AuthHelper.java**（继承时复用）

```java
package com.example.api.base;

import io.restassured.response.Response;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.equalTo;

/**
 * 鉴权辅助：获取指定账号的 Bearer Token。
 */
public abstract class AuthHelper extends BaseApiTest {

    protected String loginAs(String username, String password) {
        Response res = given()
            .spec(requestSpec)
            .body(java.util.Map.of(
                "username", username,
                "password", password,
                "captcha", "0000",   // 测试环境 captcha 永远通过
                "captchaId", "test-captcha"
            ))
        .when()
            .post("/api/auth/login")
        .then()
            .statusCode(200)
            .body("code", equalTo(200))
            .extract().response();
        return "Bearer " + res.path("data.token");
    }

    protected String adminToken() { return loginAs("admin", "admin123"); }
    protected String userToken()  { return loginAs("user01", "user123"); }
}
```

---

## 5. 用例设计矩阵（每个端点至少覆盖 8 类）

| # | 用例类型 | 必含 | 验证目标 |
|---|---------|------|---------|
| 1 | **正常路径** | ✓ | 200 + 业务码 200 + 数据正确 |
| 2 | **参数校验** | ✓ | 400 + 业务码 400 + msg 明确 |
| 3 | **未鉴权** | ✓ | 401 + 业务码 401 |
| 4 | **越权** | ✓ | 403 + 业务码 403（**不返 404 避免泄露**） |
| 5 | **资源不存在** | ✓ | 404 + 业务码 404 |
| 6 | **业务冲突** | ✓ | 409 / 423 / 422 视业务而定 |
| 7 | **边界值** | ✓ | min / max / 空串 / null / 超长 |
| 8 | **幂等性** | 视情况 | 重复请求应得相同结果（PUT / DELETE） |

---

## 6. 三端点完整示例

### 6.1 LoginApiTest.java（登录端点，覆盖 5 次密码错误锁定）

```java
package com.example.api.login;

import com.example.api.base.BaseApiTest;
import io.restassured.response.Response;
import org.junit.jupiter.api.*;
import org.springframework.test.annotation.DirtiesContext;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@DisplayName("登录接口集成测试")
class LoginApiTest extends BaseApiTest {

    @Test
    @DisplayName("正常登录：admin 账号 + 正确密码 → 200 + 返回 token")
    void login_success() {
        given().spec(requestSpec)
            .body(java.util.Map.of(
                "username", "admin", "password", "admin123",
                "captcha", "0000", "captchaId", "test"
            ))
        .when().post("/api/auth/login")
        .then()
            .statusCode(200)
            .body("code", equalTo(200))
            .body("data.token", notNullValue())
            .body("data.expiresIn", greaterThan(0));
    }

    @Test
    @DisplayName("密码错误 5 次 → 第 6 次返回 423 Locked")
    @DirtiesContext
    void login_locked_after_5_failures() {
        for (int i = 0; i < 5; i++) {
            given().spec(requestSpec)
                .body(java.util.Map.of(
                    "username", "admin", "password", "wrong_pwd",
                    "captcha", "0000", "captchaId", "test"
                ))
            .when().post("/api/auth/login")
            .then().statusCode(200).body("code", equalTo(401));
        }
        Response res = given().spec(requestSpec)
            .body(java.util.Map.of(
                "username", "admin", "password", "wrong_pwd",
                "captcha", "0000", "captchaId", "test"
            ))
        .when().post("/api/auth/login")
        .then()
            .statusCode(423)
            .body("code", equalTo(423))
            .extract().response();
        Assertions.assertTrue(res.path("msg").toString().contains("锁定"));
    }

    @Test
    @DisplayName("参数缺失：username 为空 → 400")
    void login_missing_username() {
        given().spec(requestSpec)
            .body(java.util.Map.of(
                "password", "admin123", "captcha", "0000"
            ))
        .when().post("/api/auth/login")
        .then()
            .statusCode(400)
            .body("code", equalTo(400))
            .body("msg", containsString("用户名"));
    }

    @Test
    @DisplayName("账号不存在 → 401 + 模糊提示（不暴露账号是否存在）")
    void login_user_not_exist() {
        given().spec(requestSpec)
            .body(java.util.Map.of(
                "username", "ghost_user_xxx", "password", "any",
                "captcha", "0000", "captchaId", "test"
            ))
        .when().post("/api/auth/login")
        .then()
            .statusCode(200)
            .body("code", equalTo(401))
            .body("msg", anyOf(containsString("账号"), containsString("密码")));
    }

    @Test
    @DisplayName("Captcha 错误 → 400")
    void login_wrong_captcha() {
        given().spec(requestSpec)
            .body(java.util.Map.of(
                "username", "admin", "password", "admin123",
                "captcha", "9999", "captchaId", "test"
            ))
        .when().post("/api/auth/login")
        .then()
            .statusCode(400)
            .body("code", equalTo(400));
    }
}
```

### 6.2 UserApiTest.java（用户端点，覆盖 CRUD + 越权）

```java
package com.example.api.user;

import com.example.api.base.AuthHelper;
import org.junit.jupiter.api.*;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@DisplayName("用户管理接口集成测试")
class UserApiTest extends AuthHelper {

    @Test
    @DisplayName("分页查询用户列表：ADMIN → 200 + total/list 结构")
    void list_users_as_admin() {
        given().spec(requestSpec)
            .header("Authorization", adminToken())
            .queryParam("pageNum", 1)
            .queryParam("pageSize", 10)
        .when().get("/api/users")
        .then()
            .statusCode(200)
            .body("code", equalTo(200))
            .body("data.total", greaterThanOrEqualTo(0))
            .body("data.list", hasSize(greaterThanOrEqualTo(0)));
    }

    @Test
    @DisplayName("分页查询：USER 角色越权 → 403")
    void list_users_as_user_forbidden() {
        given().spec(requestSpec)
            .header("Authorization", userToken())
            .queryParam("pageNum", 1)
            .queryParam("pageSize", 10)
        .when().get("/api/users")
        .then()
            .statusCode(200)
            .body("code", equalTo(403));
    }

    @Test
    @DisplayName("未携带 Token → 401")
    void list_users_unauthorized() {
        given().spec(requestSpec)
        .when().get("/api/users")
        .then()
            .statusCode(200)
            .body("code", equalTo(401));
    }

    @Test
    @DisplayName("创建用户：用户名重复 → 业务码 409")
    void create_user_duplicate() {
        given().spec(requestSpec)
            .header("Authorization", adminToken())
            .body(java.util.Map.of(
                "username", "admin",
                "password", "Test123!",
                "nickname", "测试",
                "email", "test@dup.com",
                "roleIds", java.util.List.of(2L)
            ))
        .when().post("/api/users")
        .then()
            .statusCode(200)
            .body("code", equalTo(409))
            .body("msg", containsString("用户名已存在"));
    }

    @Test
    @DisplayName("创建用户：邮箱格式错误 → 400")
    void create_user_invalid_email() {
        given().spec(requestSpec)
            .header("Authorization", adminToken())
            .body(java.util.Map.of(
                "username", "new_" + System.currentTimeMillis(),
                "password", "Test123!",
                "nickname", "测试",
                "email", "not-an-email",
                "roleIds", java.util.List.of(2L)
            ))
        .when().post("/api/users")
        .then()
            .statusCode(400)
            .body("code", equalTo(400))
            .body("msg", containsString("邮箱"));
    }

    @Test
    @DisplayName("创建用户：弱密码（< 8 位）→ 400")
    void create_user_weak_password() {
        given().spec(requestSpec)
            .header("Authorization", adminToken())
            .body(java.util.Map.of(
                "username", "new_" + System.currentTimeMillis(),
                "password", "123",
                "nickname", "测试",
                "email", "ok@example.com",
                "roleIds", java.util.List.of(2L)
            ))
        .when().post("/api/users")
        .then()
            .statusCode(400)
            .body("code", equalTo(400));
    }

    @Test
    @DisplayName("更新用户：用户改自己昵称 → 200")
    void update_my_profile() {
        given().spec(requestSpec)
            .header("Authorization", userToken())
            .body(java.util.Map.of("nickname", "新昵称"))
        .when().put("/api/users/me")
        .then()
            .statusCode(200)
            .body("code", equalTo(200))
            .body("data.nickname", equalTo("新昵称"));
    }

    @Test
    @DisplayName("删除用户：USER 越权 → 403")
    void delete_user_forbidden() {
        given().spec(requestSpec)
            .header("Authorization", userToken())
        .when().delete("/api/users/1")
        .then()
            .statusCode(200)
            .body("code", equalTo(403));
    }

    @Test
    @DisplayName("删除用户：ADMIN 删除不存在用户 → 404")
    void delete_user_not_found() {
        given().spec(requestSpec)
            .header("Authorization", adminToken())
        .when().delete("/api/users/999999")
        .then()
            .statusCode(200)
            .body("code", equalTo(404));
    }
}
```

### 6.3 RoleApiTest.java（角色端点，覆盖绑定 + 越权）

```java
package com.example.api.role;

import com.example.api.base.AuthHelper;
import org.junit.jupiter.api.*;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@DisplayName("角色管理接口集成测试")
class RoleApiTest extends AuthHelper {

    @Test
    @DisplayName("查询角色列表：USER 可读 → 200")
    void list_roles_as_user() {
        given().spec(requestSpec)
            .header("Authorization", userToken())
        .when().get("/api/roles")
        .then()
            .statusCode(200)
            .body("code", equalTo(200))
            .body("data", hasSize(greaterThan(0)));
    }

    @Test
    @DisplayName("创建角色：USER 越权 → 403")
    void create_role_forbidden() {
        given().spec(requestSpec)
            .header("Authorization", userToken())
            .body(java.util.Map.of(
                "code", "TESTER",
                "name", "测试员",
                "permissionIds", java.util.List.of(1L)
            ))
        .when().post("/api/roles")
        .then()
            .statusCode(200)
            .body("code", equalTo(403));
    }

    @Test
    @DisplayName("创建角色：ADMIN + 重复 code → 409")
    void create_role_duplicate_code() {
        given().spec(requestSpec)
            .header("Authorization", adminToken())
            .body(java.util.Map.of(
                "code", "ADMIN",
                "name", "管理员",
                "permissionIds", java.util.List.of()
            ))
        .when().post("/api/roles")
        .then()
            .statusCode(200)
            .body("code", equalTo(409));
    }

    @Test
    @DisplayName("绑定角色：USER 改自己 → 403（仅 ADMIN 可改）")
    void bind_role_self_forbidden() {
        given().spec(requestSpec)
            .header("Authorization", userToken())
            .body(java.util.Map.of("roleIds", java.util.List.of(2L)))
        .when().put("/api/users/me/roles")
        .then()
            .statusCode(200)
            .body("code", equalTo(403));
    }

    @Test
    @DisplayName("绑定角色：ADMIN 绑定不存在角色 → 404")
    void bind_role_not_found() {
        given().spec(requestSpec)
            .header("Authorization", adminToken())
            .body(java.util.Map.of("roleIds", java.util.List.of(99999L)))
        .when().put("/api/users/2/roles")
        .then()
            .statusCode(200)
            .body("code", equalTo(404));
    }

    @Test
    @DisplayName("删除角色：被引用时（admin）→ 409 + 业务提示")
    void delete_role_in_use() {
        given().spec(requestSpec)
            .header("Authorization", adminToken())
        .when().delete("/api/roles/1")
        .then()
            .statusCode(200)
            .body("code", equalTo(409))
            .body("msg", containsString("正在被"));
    }

    @Test
    @DisplayName("删除角色：ADMIN 越权删 → 403（仅 SUPER_ADMIN 可删）")
    void delete_role_as_admin_forbidden() {
        given().spec(requestSpec)
            .header("Authorization", adminToken())
        .when().delete("/api/roles/2")
        .then()
            .statusCode(200)
            .body("code", equalTo(403));
    }
}
```

---

## 7. 数据准备与清理

**策略 A：每个测试方法前 TRUNCATE（推荐，隔离最强）**

```java
@BeforeEach
void cleanDb() {
    JdbcTemplate jdbc = SpringContextHolder.getBean(JdbcTemplate.class);
    jdbc.execute("SET FOREIGN_KEY_CHECKS = 0");
    jdbc.execute("TRUNCATE TABLE sys_user_role");
    jdbc.execute("TRUNCATE TABLE sys_user");
    jdbc.execute("TRUNCATE TABLE sys_role");
    jdbc.execute("SET FOREIGN_KEY_CHECKS = 1");
    TestDataFactory.seedBasicData(jdbc);  // 重插 admin / user01
}
```

**策略 B：每测试类 `@Transactional` + 回滚（速度最快，但测不到事务提交后的逻辑）**

```java
@SpringBootTest
@Transactional
@Rollback
class UserApiTest { ... }
```

> **推荐组合**：策略 A 兜底（兼容性最强）+ 关键事务提交逻辑用策略 B 单独测。

---

## 8. 执行命令与报告输出

```bash
# 跑全部接口测试
mvnw test -Dtest='*ApiTest'

# 跑单个端点
mvnw test -Dtest='LoginApiTest'

# 跑单个方法
mvnw test -Dtest='LoginApiTest#login_locked_after_5_failures'

# 生成 Allure 报告
mvnw test -Dtest='*ApiTest' -Dallure.results.directory=./target/allure-results
mvnw allure:serve  # 本地浏览器查看

# Surefire 报告位置
# backend/target/surefire-reports/*.xml
# backend/target/surefire-reports/*.html
```

**测试输出标准**：

```
[INFO] Tests run: 28, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

---

## 9. CI 接入（GitHub Actions 片段）

```yaml
name: api-integration-test

on: [pull_request]

jobs:
  api-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '17'

      - name: Run API integration tests
        run: ./mvnw test -Dtest='*ApiTest' -Dsurefire.useFile=false

      - name: Upload surefire reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: surefire-reports
          path: backend/target/surefire-reports/
```

---

## 10. 禁止事项

| 禁止 | 原因 |
|------|------|
| ❌ `@MockBean UserMapper` 跳过真实 DB | 违反「真实库交互」根本要求，测试失真 |
| ❌ 用 H2 内存数据库替代 MySQL | 语法差异（`IFNULL`、分页 `LIMIT`、日期函数）会导致生产 bug 漏测 |
| ❌ 测试中 `Thread.sleep(1000)` 等待异步 | 用 `Awaitility` 或同步阻塞 + CountDownLatch |
| ❌ 跨测试方法共享可变状态（`static` 字段赋值） | 顺序依赖导致 CI 偶发失败 |
| ❌ 用生产库 URL 跑接口测试 | 误删数据，必须 testcontainers 隔离 |
| ❌ MySQL 镜像用 `latest` tag | 版本漂移导致「本地能跑 CI 挂」 |
| ❌ 测试方法不写 `@DisplayName` | Allure 报告显示「test1()」无法定位失败 |
| ❌ 越权测试用同账号 + 不同权限角色 | 必须用**第二个独立用户**的 token |
| ❌ 测试数据硬编码 `id=1` 等固定值 | 用 `findByUsername` 动态获取 |
| ❌ 不重置 Redis 失败计数就连续跑锁定测试 | 锁定状态污染后续用例 |
