# 数据库规范

## 命名
- 表名：snake_case，复数形式（`user_roles`）
- 字段名：snake_case（`create_time`、`user_id`）
- 主键：统一用 `id`，BIGINT 自增
- 必备字段：`id`、`create_time`、`update_time`、`deleted`（逻辑删除）

## SQL 规范
- 关键字大写：`SELECT`、`FROM`、`WHERE`
- 每个字段换行对齐
- 禁止用 `SELECT *`，必须明确字段
- 索引命名：`idx_表名_字段名`（`idx_user_email`）

## 迁移脚本
- 脚本命名：`V{版本号}__{描述}.sql`（`V1.0__init_user_table.sql`）
- 脚本放 `db/migration/` 目录
- 每次变更一个脚本，不要修改已执行的脚本

## full.sql（完整建库脚本）
- `db/full.sql` 包含**所有表结构 + 索引 + INSERT 初始化数据**，可一次性执行完成整个数据库的创建
- 只用 `CREATE TABLE IF NOT EXISTS` + `INSERT`，**禁止出现 `ALTER`、`UPDATE`、`DROP`**
- 每次新增或修改表结构/字段后，同步更新 `full.sql`，保证它始终是最新完整版
- 用途：新环境部署、数据库重建时直接执行这一个文件即可

## 示例
```sql
CREATE TABLE IF NOT EXISTS `user_info` (
    `id`          BIGINT       NOT NULL AUTO_INCREMENT,
    `username`    VARCHAR(50)  NOT NULL COMMENT '用户名',
    `email`       VARCHAR(100) NOT NULL COMMENT '邮箱',
    `create_time` DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `update_time` DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    `deleted`     TINYINT      NOT NULL DEFAULT 0 COMMENT '逻辑删除 0-未删 1-已删',
    PRIMARY KEY (`id`),
    UNIQUE KEY `idx_user_email` (`email`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户信息表';

-- 初始化数据（5条符合业务的示例数据）
INSERT INTO `user_info` (`username`, `email`) VALUES
('张三', 'zhangsan@example.com'),
('李四', 'lisi@example.com'),
('王五', 'wangwu@example.com'),
('赵六', 'zhaoliu@example.com'),
('孙七', 'sunqi@example.com');
```

## 初始化数据规则
- 每张表必须附带 `INSERT` 语句，提供 5 条符合业务场景的示例数据
- 数据要真实合理，不要用 test1/test2 这种无意义数据
- 修改字段后，同步更新对应的 INSERT 语句，保持数据与表结构一致

## 自动执行规则
- 任何涉及数据库的改动（建表、改字段、加索引等），完成后**必须主动执行 SQL 更新数据库**
- 执行顺序：先执行 DDL（建表/改表），再执行 DML（INSERT 数据）
