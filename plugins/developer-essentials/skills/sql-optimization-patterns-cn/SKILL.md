---
name: sql-optimization-patterns-cn
description: 掌握 SQL 查询优化、索引策略和 EXPLAIN 分析,显著提升数据库性能,消除慢查询。适用于调试慢查询、设计数据库架构或优化应用性能。
---

# SQL 优化模式

通过系统性优化、正确索引和查询计划分析,将慢速数据库查询转换为闪电般快速的操作。

## 使用场景

- 调试慢查询
- 设计高性能数据库架构
- 优化应用响应时间
- 降低数据库负载和成本
- 提升数据增长时的可扩展性
- 分析 EXPLAIN 查询计划
- 实现高效索引
- 解决 N+1 查询问题

## 核心概念

### 1. 查询执行计划 (EXPLAIN)

理解 EXPLAIN 输出是优化的基础。

**PostgreSQL EXPLAIN:**
```sql
-- 基础 explain
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- 带实际执行统计
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user@example.com';

-- 详细输出模式
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT u.*, o.order_total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.created_at > NOW() - INTERVAL '30 days';
```

**关键指标:**
- **Seq Scan**: 全表扫描 (大表通常很慢)
- **Index Scan**: 使用索引 (良好)
- **Index Only Scan**: 仅使用索引不触碰表 (最佳)
- **Nested Loop**: 连接方法 (适合小数据集)
- **Hash Join**: 连接方法 (适合大数据集)
- **Merge Join**: 连接方法 (适合已排序数据)
- **Cost**: 预估查询成本 (越低越好)
- **Rows**: 预估返回行数
- **Actual Time**: 实际执行时间

### 2. 索引策略

索引是最强大的优化工具。

**索引类型:**
- **B-Tree**: 默认类型,适合等值和范围查询
- **Hash**: 仅用于等值 (=) 比较
- **GIN**: 全文搜索、数组查询、JSONB
- **GiST**: 几何数据、全文搜索
- **BRIN**: 块范围索引,适用于有相关性的超大表

```sql
-- 标准 B-Tree 索引
CREATE INDEX idx_users_email ON users(email);

-- 复合索引 (顺序很重要!)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- 部分索引 (仅索引行的子集)
CREATE INDEX idx_active_users ON users(email)
WHERE status = 'active';

-- 表达式索引
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- 覆盖索引 (包含额外列)
CREATE INDEX idx_users_email_covering ON users(email)
INCLUDE (name, created_at);

-- 全文搜索索引
CREATE INDEX idx_posts_search ON posts
USING GIN(to_tsvector('english', title || ' ' || body));

-- JSONB 索引
CREATE INDEX idx_metadata ON events USING GIN(metadata);
```

### 3. 查询优化模式

**避免 SELECT \*:**
```sql
-- 差: 获取不必要的列
SELECT * FROM users WHERE id = 123;

-- 好: 仅获取需要的列
SELECT id, email, name FROM users WHERE id = 123;
```

**高效使用 WHERE 子句:**
```sql
-- 差: 函数阻止索引使用
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- 好: 创建函数索引或使用精确匹配
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
-- 然后:
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- 或存储标准化数据
SELECT * FROM users WHERE email = 'user@example.com';
```

**优化 JOIN:**
```sql
-- 差: 先生成笛卡尔积再过滤
SELECT u.name, o.total
FROM users u, orders o
WHERE u.id = o.user_id AND u.created_at > '2024-01-01';

-- 好: 连接前过滤
SELECT u.name, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01';

-- 更好: 过滤两个表
SELECT u.name, o.total
FROM (SELECT * FROM users WHERE created_at > '2024-01-01') u
JOIN orders o ON u.id = o.user_id;
```

## 优化模式

### 模式 1: 消除 N+1 查询

**问题: N+1 查询反模式**
```python
# 差: 执行 N+1 条查询
users = db.query("SELECT * FROM users LIMIT 10")
for user in users:
    orders = db.query("SELECT * FROM orders WHERE user_id = ?", user.id)
    # 处理订单
```

**解决方案: 使用 JOIN 或批量加载**
```sql
-- 方案 1: JOIN
SELECT
    u.id, u.name,
    o.id as order_id, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.id IN (1, 2, 3, 4, 5);

-- 方案 2: 批量查询
SELECT * FROM orders
WHERE user_id IN (1, 2, 3, 4, 5);
```

```python
# 好: 使用 JOIN 或批量加载的单条查询
# 使用 JOIN
results = db.query("""
    SELECT u.id, u.name, o.id as order_id, o.total
    FROM users u
    LEFT JOIN orders o ON u.id = o.user_id
    WHERE u.id IN (1, 2, 3, 4, 5)
""")

# 或批量加载
users = db.query("SELECT * FROM users LIMIT 10")
user_ids = [u.id for u in users]
orders = db.query(
    "SELECT * FROM orders WHERE user_id IN (?)",
    user_ids
)
# 按 user_id 分组订单
orders_by_user = {}
for order in orders:
    orders_by_user.setdefault(order.user_id, []).append(order)
```

### 模式 2: 优化分页

**差: 大表上的 OFFSET**
```sql
-- 大偏移量时很慢
SELECT * FROM users
ORDER BY created_at DESC
LIMIT 20 OFFSET 100000;  -- 非常慢!
```

**好: 基于游标的分页**
```sql
-- 更快: 使用游标 (最后看到的 ID)
SELECT * FROM users
WHERE created_at < '2024-01-15 10:30:00'  -- 上次游标位置
ORDER BY created_at DESC
LIMIT 20;

-- 带复合排序
SELECT * FROM users
WHERE (created_at, id) < ('2024-01-15 10:30:00', 12345)
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- 需要索引
CREATE INDEX idx_users_cursor ON users(created_at DESC, id DESC);
```

### 模式 3: 高效聚合

**优化 COUNT 查询:**
```sql
-- 差: 统计所有行
SELECT COUNT(*) FROM orders;  -- 大表上很慢

-- 好: 使用估算值获取近似计数
SELECT reltuples::bigint AS estimate
FROM pg_class
WHERE relname = 'orders';

-- 好: 统计前先过滤
SELECT COUNT(*) FROM orders
WHERE created_at > NOW() - INTERVAL '7 days';

-- 更好: 使用仅索引扫描
CREATE INDEX idx_orders_created ON orders(created_at);
SELECT COUNT(*) FROM orders
WHERE created_at > NOW() - INTERVAL '7 days';
```

**优化 GROUP BY:**
```sql
-- 差: 先分组再过滤
SELECT user_id, COUNT(*) as order_count
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 10;

-- 更好: 先过滤再分组 (如果可能)
SELECT user_id, COUNT(*) as order_count
FROM orders
WHERE status = 'completed'
GROUP BY user_id
HAVING COUNT(*) > 10;

-- 最佳: 使用覆盖索引
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

### 模式 4: 子查询优化

**转换相关子查询:**
```sql
-- 差: 相关子查询 (为每行执行)
SELECT u.name, u.email,
    (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) as order_count
FROM users u;

-- 好: 使用 JOIN 聚合
SELECT u.name, u.email, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name, u.email;

-- 更好: 使用窗口函数
SELECT DISTINCT ON (u.id)
    u.name, u.email,
    COUNT(o.id) OVER (PARTITION BY u.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
```

**使用 CTE 提升清晰度:**
```sql
-- 使用公用表表达式
WITH recent_users AS (
    SELECT id, name, email
    FROM users
    WHERE created_at > NOW() - INTERVAL '30 days'
),
user_order_counts AS (
    SELECT user_id, COUNT(*) as order_count
    FROM orders
    WHERE created_at > NOW() - INTERVAL '30 days'
    GROUP BY user_id
)
SELECT ru.name, ru.email, COALESCE(uoc.order_count, 0) as orders
FROM recent_users ru
LEFT JOIN user_order_counts uoc ON ru.id = uoc.user_id;
```

### 模式 5: 批量操作

**批量 INSERT:**
```sql
-- 差: 多次单独插入
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
INSERT INTO users (name, email) VALUES ('Bob', 'bob@example.com');
INSERT INTO users (name, email) VALUES ('Carol', 'carol@example.com');

-- 好: 批量插入
INSERT INTO users (name, email) VALUES
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com'),
    ('Carol', 'carol@example.com');

-- 更好: 使用 COPY 进行批量插入 (PostgreSQL)
COPY users (name, email) FROM '/tmp/users.csv' CSV HEADER;
```

**批量 UPDATE:**
```sql
-- 差: 循环中更新
UPDATE users SET status = 'active' WHERE id = 1;
UPDATE users SET status = 'active' WHERE id = 2;
-- ... 对多个 ID 重复执行

-- 好: 使用 IN 子句的单条 UPDATE
UPDATE users
SET status = 'active'
WHERE id IN (1, 2, 3, 4, 5, ...);

-- 更好: 对大批量使用临时表
CREATE TEMP TABLE temp_user_updates (id INT, new_status VARCHAR);
INSERT INTO temp_user_updates VALUES (1, 'active'), (2, 'active'), ...;

UPDATE users u
SET status = t.new_status
FROM temp_user_updates t
WHERE u.id = t.id;
```

## 高级技巧

### 物化视图

预计算昂贵查询。

```sql
-- 创建物化视图
CREATE MATERIALIZED VIEW user_order_summary AS
SELECT
    u.id,
    u.name,
    COUNT(o.id) as total_orders,
    SUM(o.total) as total_spent,
    MAX(o.created_at) as last_order_date
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;

-- 为物化视图添加索引
CREATE INDEX idx_user_summary_spent ON user_order_summary(total_spent DESC);

-- 刷新物化视图
REFRESH MATERIALIZED VIEW user_order_summary;

-- 并发刷新 (PostgreSQL)
REFRESH MATERIALIZED VIEW CONCURRENTLY user_order_summary;

-- 查询物化视图 (非常快)
SELECT * FROM user_order_summary
WHERE total_spent > 1000
ORDER BY total_spent DESC;
```

### 分区

拆分大表以获得更好性能。

```sql
-- 按日期范围分区 (PostgreSQL)
CREATE TABLE orders (
    id SERIAL,
    user_id INT,
    total DECIMAL,
    created_at TIMESTAMP
) PARTITION BY RANGE (created_at);

-- 创建分区
CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- 查询自动使用适当的分区
SELECT * FROM orders
WHERE created_at BETWEEN '2024-02-01' AND '2024-02-28';
-- 仅扫描 orders_2024_q1 分区
```

### 查询提示和优化

```sql
-- 强制使用索引 (MySQL)
SELECT * FROM users
USE INDEX (idx_users_email)
WHERE email = 'user@example.com';

-- 并行查询 (PostgreSQL)
SET max_parallel_workers_per_gather = 4;
SELECT * FROM large_table WHERE condition;

-- 连接提示 (PostgreSQL)
SET enable_nestloop = OFF;  -- 强制使用 hash 或 merge join
```

## 最佳实践

1. **选择性索引**: 过多索引会降低写入速度
2. **监控查询性能**: 使用慢查询日志
3. **保持统计更新**: 定期运行 ANALYZE
4. **使用合适的数据类型**: 更小的类型 = 更好的性能
5. **适度规范化**: 平衡规范化与性能
6. **缓存频繁访问的数据**: 使用应用级缓存
7. **连接池**: 重用数据库连接
8. **定期维护**: VACUUM、ANALYZE、重建索引

```sql
-- 更新统计信息
ANALYZE users;
ANALYZE VERBOSE orders;

-- 清理 (PostgreSQL)
VACUUM ANALYZE users;
VACUUM FULL users;  -- 回收空间 (锁定表)

-- 重建索引
REINDEX INDEX idx_users_email;
REINDEX TABLE users;
```

## 常见陷阱

- **过度索引**: 每个索引都会降低 INSERT/UPDATE/DELETE 速度
- **未使用的索引**: 浪费空间并降低写入速度
- **缺少索引**: 慢查询、全表扫描
- **隐式类型转换**: 阻止索引使用
- **OR 条件**: 无法高效使用索引
- **LIKE 前导通配符**: `LIKE '%abc'` 无法使用索引
- **WHERE 中的函数**: 阻止索引使用,除非存在函数索引

## 监控查询

```sql
-- 查找慢查询 (PostgreSQL)
SELECT query, calls, total_time, mean_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- 查找缺失的索引 (PostgreSQL)
SELECT
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    seq_tup_read / seq_scan AS avg_seq_tup_read
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC
LIMIT 10;

-- 查找未使用的索引 (PostgreSQL)
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

## 资源

- **references/postgres-optimization-guide.md**: PostgreSQL 专用优化
- **references/mysql-optimization-guide.md**: MySQL/MariaDB 优化
- **references/query-plan-analysis.md**: EXPLAIN 计划深度解析
- **assets/index-strategy-checklist.md**: 何时以及如何创建索引
- **assets/query-optimization-checklist.md**: 逐步优化指南
- **scripts/analyze-slow-queries.sql**: 识别数据库中的慢查询
- **scripts/index-recommendations.sql**: 生成索引建议
