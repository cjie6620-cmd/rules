# ===== SQL 优化、索引设计规范 =====

> 与 dba.md 配合使用，dba.md 管命名和结构，本文件管性能和优化

---

## 一、EXPLAIN 执行计划解读

**任何涉及性能的 SQL 优化，第一步都是用 EXPLAIN 分析执行计划。**

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 1001 AND status = 1 ORDER BY create_time DESC;
```

### 关键字段速查

| 字段 | 含义 | 优化目标 |
|------|------|---------|
| **type** | 访问类型 | 至少 `ref`，最好 `const` / `eq_ref` |
| **key** | 实际使用的索引 | 不能为 NULL（没用索引） |
| **rows** | 预估扫描行数 | 越小越好 |
| **Extra** | 额外信息 | 避免 `Using filesort`、`Using temporary` |

### type 访问类型（从好到差）

| 类型 | 含义 | 示例 |
|------|------|------|
| `system` | 表只有一行 | 系统表 |
| `const` | 主键/唯一索引等值查询 | `WHERE id = 1` |
| `eq_ref` | 关联查询用主键/唯一索引 | `JOIN ON a.id = b.user_id` |
| `ref` | 非唯一索引等值查询 | `WHERE user_id = 1001` |
| `range` | 索引范围查询 | `WHERE price > 100` |
| `index` | 全索引扫描 | 覆盖索引但无 WHERE 条件 |
| **`ALL`** | **全表扫描** | **必须优化！** |

### Extra 字段常见值

| Extra | 含义 | 是否需要优化 |
|-------|------|-------------|
| `Using index` | 覆盖索引（好） | 不需要 |
| `Using where` | Server 层过滤 | 看情况 |
| `Using index condition` | 索引下推（好） | 不需要 |
| **`Using filesort`** | 额外排序 | **需要优化**（排序字段没走索引） |
| **`Using temporary`** | 使用临时表 | **需要优化**（GROUP BY 字段没走索引） |
| **`Using join buffer`** | 关联查询没用索引 | **需要优化** |

---

## 二、慢查询治理流程

### 1. 开启慢查询日志

```sql
-- 查看当前配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志（线上建议阈值 200ms）
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.2;
SET GLOBAL log_queries_not_using_indexes = ON;  -- 记录未使用索引的查询
```

### 2. 分析慢查询

```bash
# 使用 mysqldumpslow 分析（MySQL 自带）
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# 或使用 pt-query-digest（Percona Toolkit，更强大）
pt-query-digest /var/log/mysql/slow.log
```

### 3. 优化流程

```
发现慢查询 → EXPLAIN 分析 → 定位瓶颈 → 优化 → 验证 → 上线监控
```

**慢查询阈值建议**：
- 线上 > 200ms：需要优化
- 线上 > 1s：必须优化
- 线上 > 5s：紧急优化

---

## 三、索引设计原则

### 1. 联合索引遵循最左前缀

```sql
-- 联合索引：idx_user_status_time (user_id, status, create_time)

-- ✅ 走索引（命中最左前缀）
WHERE user_id = 1001
WHERE user_id = 1001 AND status = 1
WHERE user_id = 1001 AND status = 1 AND create_time > '2024-01-01'

-- ✅ 走索引（MySQL 优化器会调整 WHERE 顺序）
WHERE status = 1 AND user_id = 1001  -- 等效于 WHERE user_id = 1001 AND status = 1

-- ❌ 不走索引（跳过了 user_id）
WHERE status = 1
WHERE status = 1 AND create_time > '2024-01-01'

-- ⚠️ 只走部分索引
WHERE user_id = 1001 AND create_time > '2024-01-01'  -- 只用了 user_id，create_time 没走索引（中间的 status 被跳过了）
```

### 2. 区分度高的字段放前面

```sql
-- user_id 区分度高（几万个用户），status 区分度低（只有几个值）

-- 推荐：区分度高的在前
idx_user_status (user_id, status)

-- 不推荐：区分度低的在前
idx_status_user (status, user_id)  -- 过滤效果差
```

**查看字段区分度**：

```sql
SELECT
    COUNT(DISTINCT user_id) / COUNT(*) AS user_id_selectivity,
    COUNT(DISTINCT status) / COUNT(*) AS status_selectivity
FROM orders;
-- 区分度 > 0.1 的字段适合建索引
```

### 3. 覆盖索引减少回表

```sql
-- 覆盖索引：查询的字段都在索引中，不需要回表查主键索引

-- 联合索引：idx_user_status_time (user_id, status, create_time)

-- ✅ 覆盖索引（SELECT 的字段都在索引中）
SELECT user_id, status, create_time FROM orders WHERE user_id = 1001;

-- ❌ 不是覆盖索引（amount 不在索引中，需要回表）
SELECT user_id, status, create_time, amount FROM orders WHERE user_id = 1001;

-- 使用 EXPLAIN 验证：Extra 显示 "Using index" 说明覆盖索引生效
```

### 4. 索引失效场景（必须避免）

| 场景 | 示例 | 原因 | 解决方案 |
|------|------|------|---------|
| **函数/计算** | `WHERE YEAR(create_time) = 2024` | 函数破坏索引有序性 | `WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01'` |
| **隐式类型转换** | `WHERE phone = 13800138000`（phone 是 VARCHAR） | MySQL 隐式转换导致索引失效 | `WHERE phone = '13800138000'` |
| **LIKE 左模糊** | `WHERE name LIKE '%张'` | 前缀不确定无法用索引 | `WHERE name LIKE '张%'`（右模糊走索引） |
| **OR 条件** | `WHERE user_id = 1 OR status = 1` | 两个字段各走各的索引 | 改用 `UNION ALL` 分开查询 |
| **!= / NOT IN / NOT EXISTS** | `WHERE status != 0` | 不等于无法精确定位 | 确认是否有其他索引可用，或改写为 `IN (1,2,3)` |
| **IS NULL / IS NOT NULL** | `WHERE deleted IS NULL` | 视数据分布而定 | 确保 NULL 值占比少，或用默认值替代 NULL |
| **字符集不一致** | JOIN 两个表字符集不同 | 无法使用索引 | 统一使用 `utf8mb4` |

---

## 四、SQL 优化速查

### 1. 批量插入

```sql
-- ❌ 逐条插入（1000 次网络往返）
INSERT INTO orders (user_id, amount) VALUES (1, 100);
INSERT INTO orders (user_id, amount) VALUES (2, 200);
...

-- ✅ 批量插入（1 次网络往返）
INSERT INTO orders (user_id, amount) VALUES
    (1, 100),
    (2, 200),
    (3, 300),
    ...;
-- 每批 500~1000 条，不要一次插入超过 5000 条
```

### 2. 分页优化

```sql
-- ❌ 深分页（偏移量越大越慢）
SELECT * FROM orders ORDER BY id LIMIT 1000000, 20;
-- MySQL 需要扫描 1000020 行，丢弃前 1000000 行

-- ✅ 方案一：延迟关联（先查主键再关联）
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 1000000, 20
) t ON o.id = t.id;

-- ✅ 方案二：游标分页（知道上一页最后一条的 ID）
SELECT * FROM orders WHERE id > #{lastId} ORDER BY id LIMIT 20;
-- 必须有 ORDER BY id 保证顺序稳定
```

### 3. COUNT 优化

```sql
-- 精确计数
SELECT COUNT(*) FROM orders WHERE user_id = 1001;
-- 用覆盖索引时 COUNT(*) 很快，非覆盖索引时看数据量

-- 近似值（数据量大且允许误差时）
SHOW TABLE STATUS LIKE 'orders';  -- rows 字段是近似值

-- 分页总数优化：先判断是否需要精确值
-- 如果数据量 > 100 万，考虑只返回 "是否有下一页"
SELECT COUNT(*) FROM (SELECT 1 FROM orders WHERE user_id = 1001 LIMIT 10001) t;
```

### 4. UPDATE / DELETE 安全规范

```sql
-- ❌ 危险：无 WHERE 的全表更新
UPDATE orders SET status = 1;

-- ❌ 危险：无 LIMIT 的大批量更新（长时间锁表）
UPDATE orders SET status = 1 WHERE create_time < '2023-01-01';

-- ✅ 加 LIMIT 分批更新
UPDATE orders SET status = 1 WHERE create_time < '2023-01-01' LIMIT 1000;
-- 循环执行直到影响行数为 0

-- ✅ 删除同理
DELETE FROM logs WHERE create_time < '2023-01-01' LIMIT 1000;
```

### 5. JOIN 优化

```sql
-- ❌ 被驱动表没索引（笛卡尔积，性能灾难）
SELECT * FROM orders o JOIN users u ON o.user_name = u.name;
-- 如果 user_name 没索引，每条 orders 记录都全表扫描 users

-- ✅ 被驱动表的关联字段必须有索引
-- JOIN 的顺序：小表驱动大表（数据量小的放前面）

-- ❌ SELECT *（回表浪费）
SELECT * FROM orders o JOIN users u ON o.user_id = u.id;

-- ✅ 只查需要的字段
SELECT o.order_no, o.amount, u.name FROM orders o JOIN users u ON o.user_id = u.id;
```

---

## 五、索引设计检查清单

新建索引前检查：

- [ ] 是否遵循最左前缀（联合索引字段顺序是否正确）
- [ ] 区分度是否足够（`COUNT(DISTINCT col) / COUNT(*) > 0.1`）
- [ ] 是否覆盖了高频查询的 WHERE / ORDER BY / GROUP BY
- [ ] 单表索引数量是否合理（建议不超过 5~6 个）
- [ ] 是否避免了冗余索引（`idx_a` 和 `idx_a_b` 中，`idx_a` 是冗余的）
- [ ] 是否避免了重复索引（`(a, b)` 和 `(b, a)` 是不同索引，不要重复建）

```sql
-- 查看表上所有索引
SHOW INDEX FROM orders;

-- 查看冗余索引（pt-duplicate-key-checker）
pt-duplicate-key-checker --host=localhost --user=root --password=xxx
```

---

## 六、禁止事项

- **禁止无 WHERE 的 UPDATE / DELETE**（全表更新/删除是灾难性操作）
- **禁止生产环境手动执行 DDL**（必须走迁移脚本）
- **禁止无索引的大表查询**（EXPLAIN type=ALL 的查询必须优化）
- **禁止深分页用 LIMIT offset, size**（offset > 10000 改用游标分页）
- **禁止在 WHERE 条件中对索引字段使用函数**
- **禁止隐式类型转换**（VARCHAR 字段必须用引号）
- **禁止 SELECT ***（明确字段列表，避免回表浪费）
- **禁止大批量操作不加 LIMIT**（UPDATE / DELETE 加 LIMIT 分批执行）
- **禁止不看 EXPLAIN 就建索引**（先分析，再建索引，最后验证）
- **禁止单表索引超过 6 个**（索引太多影响写入性能）
