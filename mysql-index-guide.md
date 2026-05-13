# MySQL 索引与复杂应用

> 面向 PHP 开发者，从原理到实战

---

## 目录

1. [索引基础原理](#1-索引基础原理)
2. [索引类型](#2-索引类型)
3. [B+Tree 深入](#3-btree-深入)
4. [EXPLAIN 执行计划](#4-explain-执行计划)
5. [索引优化策略](#5-索引优化策略)
6. [多表关联优化](#6-多表关联优化)
7. [分页优化](#7-分页优化)
8. [排序与分组优化](#8-排序与分组优化)
9. [复杂查询实战](#9-复杂查询实战)
10. [慢查询定位与优化](#10-慢查询定位与优化)
11. [索引失效场景全集](#11-索引失效场景全集)
12. [大表索引策略](#12-大表索引策略)
13. [主从复制与读写分离](#13-主从复制与读写分离)
14. [分库分表](#14-分库分表)
15. [全文索引](#15-全文索引)
16. [索引与锁](#16-索引与锁)

---

## 1. 索引基础原理

### 1.1 没有索引时

```sql
-- 全表扫描，1000万行 = 1000万次磁盘 I/O
SELECT * FROM users WHERE email = 'khz@example.com';
-- 逐行对比 email 字段，慢到怀疑人生
```

### 1.2 有索引时

```
索引 = 书的目录

目录（索引）                         数据页（书页）
┌──────────────────┐              ┌──────────────────┐
│ a@example.com → P1│              │ P1: user:1, Alice, ... │
│ bob@test.com  → P2│              │ P2: user:2, Bob, ...   │
│ khz@example.com→P3│              │ P3: user:5, khz, ...   │
│ zoe@mail.com  → P4│              │ P4: user:9, Zoe, ...   │
└──────────────────┘              └──────────────────┘

查询 email='khz@example.com'：
1. 在目录二分查找 → 找到 P3
2. 直接翻到 P3 页
3. 读取数据 → 完成

时间复杂度：O(log n)，千万级数据只需约 24 次查找
```

### 1.3 索引的代价

```
✅ 优点：
  - 查询速度指数级提升
  - ORDER BY / GROUP BY 避免 filesort
  - 加速 JOIN 关联

❌ 代价：
  - 写操作变慢（INSERT/UPDATE/DELETE 要维护索引）
  - 占用磁盘空间（通常是数据的 1.5-2 倍）
  - 缓存利用率下降
```

---

## 2. 索引类型

### 2.1 主键索引（聚簇索引）

```sql
-- InnoDB 必须有一个聚簇索引
-- 数据按主键顺序物理存储

CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50),
    email VARCHAR(100)
);

-- id 是聚簇索引，整行数据和 id 存在一起
-- 查询 id 直接拿到全部数据
SELECT * FROM users WHERE id = 100;  -- 一次查找即可
```

### 2.2 唯一索引

```sql
-- 业务唯一约束 + 查询加速
ALTER TABLE users ADD UNIQUE INDEX uk_email (email);

-- 相等的查询，找到第一条就停止
SELECT * FROM users WHERE email = 'khz@example.com';
-- 比普通索引更快：知道只有一条，不需要向后扫描
```

### 2.3 普通索引（二级索引）

```sql
-- 叶子节点存储的是主键值，不是行数据
ALTER TABLE users ADD INDEX idx_name (name);

-- 查询过程（回表）
-- 1. 在 idx_name 中找 'khz' → 拿到 id=5
-- 2. 用 id=5 去聚簇索引取整行数据
SELECT * FROM users WHERE name = 'khz';
```

### 2.4 联合索引

```sql
-- ⭐ 最重要的索引类型，80% 的优化靠它
ALTER TABLE orders ADD INDEX idx_user_status_time (user_id, status, created_at);

-- 最左前缀原则：
-- ✅ 能用到索引
WHERE user_id = 100;
WHERE user_id = 100 AND status = 'paid';
WHERE user_id = 100 AND status = 'paid' AND created_at > '2026-01-01';
WHERE user_id = 100 AND created_at > '2026-01-01';  -- 只用到 user_id

-- ❌ 用不到索引
WHERE status = 'paid';                          -- 跳过了第一列
WHERE created_at > '2026-01-01';                -- 跳过了前两列
WHERE status = 'paid' AND created_at > '2026-01-01';  -- 跳过了 user_id

-- 范围查询右边的列失效
WHERE user_id = 100 AND status > 'paid' AND created_at > '2026-01-01';
--     ✅              ✅ 范围断了        ❌ 右边失效
```

### 2.5 覆盖索引

```sql
-- 查询的所有列都在索引里，不需要回表
-- 这是最快的情况

-- 假设索引：idx_user_status_time (user_id, status, created_at)
SELECT user_id, status, created_at FROM orders
WHERE user_id = 100 AND status = 'paid';
-- ✅ 覆盖索引：需要的列都在索引中，不需要回表

-- 对比（需要回表）
SELECT * FROM orders WHERE user_id = 100;
-- ❌ 需要回表拿到 amount 等不在索引中的列

-- 覆盖索引实战：给高频查询建特定索引
ALTER TABLE orders ADD INDEX idx_user_amount (user_id, amount, created_at);

-- 这个查询就被完全覆盖了
SELECT user_id, amount, created_at
FROM orders
WHERE user_id = 100
ORDER BY created_at DESC
LIMIT 10;
```

### 2.6 前缀索引

```sql
-- 长字符串建索引，只取前 N 个字符
ALTER TABLE articles ADD INDEX idx_title_prefix (title(20));

-- 代价：无法使用覆盖索引，ORDER BY 不能走索引

-- 选择合适长度
SELECT
    COUNT(DISTINCT LEFT(title, 10)) / COUNT(*) AS sel_10,
    COUNT(DISTINCT LEFT(title, 15)) / COUNT(*) AS sel_15,
    COUNT(DISTINCT LEFT(title, 20)) / COUNT(*) AS sel_20
FROM articles;
-- 选择区分度接近 1.0 的最小长度
```

### 2.7 索引类型速查

| 类型 | 存储 | 用途 | 特点 |
|------|------|------|------|
| 聚簇索引 | 存整行数据 | 主键 | 每表只有一个 |
| 唯一索引 | 存主键值 | 唯一约束 | 找到即停止 |
| 普通索引 | 存主键值 | 通用查询 | 需要回表 |
| 联合索引 | 存主键值 | 多条件查询 | 最左前缀原则 |
| 覆盖索引 | — | 不回表查询 | 一种优化状态 |
| 前缀索引 | 存主键值 | 长字符串 | 牺牲精确度 |
| 全文索引 | 倒排索引 | 文本搜索 | 分词匹配 |

---

## 3. B+Tree 深入

### 3.1 为什么是 B+Tree

```
数据结构对比：

二叉树        → 数据量大时层级太深，I/O 次数多
红黑树        → 也是二叉树，同样的问题
B Tree       → 非叶子节点也存数据，每个节点放不了太多键
B+Tree ✅    → 只在叶子存数据，非叶子节点存更多键，树更矮
Hash         → 快但不支持范围查询、排序

B+Tree 优势：
- 矮胖：3-4 层就能存千万数据（16KB 页 × 约 1170 键/层）
  层1: 1170 个键
  层2: 1170 × 1170 = 137万个键
  层3: 137万 × 1170 = 16亿个键 → 实际 3-4 层足够
- 叶子有序：天然支持范围查询和排序
- 叶子链表：可以快速扫全表或范围
```

### 3.2 InnoDB 页结构

```
┌─────────────────────────────────────┐
│ 页头 (38B)                          │
│  - 页号、类型、上一页/下一页指针     │
├─────────────────────────────────────┤
│ User Records (用户记录)              │
│  [id=1,name='a']                    │
│  [id=3,name='c']                    │
│  [id=5,name='d']                    │
│  ...                                │
├─────────────────────────────────────┤
│ Free Space                          │
├─────────────────────────────────────┤
│ Page Directory (页目录，槽)          │
│  [槽1] → 指向记录组                 │
│  [槽2] → 指向记录组                 │
├─────────────────────────────────────┤
│ 页尾 (8B)                           │
└─────────────────────────────────────┘

InnoDB 页大小默认 16KB
查找过程：二分查找 Page Directory → 定位到槽 → 遍历组内记录
```

---

## 4. EXPLAIN 执行计划

### 4.1 基本用法

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100 AND status = 'paid';
```

```
+----+-------------+--------+------+---------------------+------+---------+-------+------+-------+
| id | select_type | table  | type | possible_keys        | key  | key_len | ref   | rows | Extra |
+----+-------------+--------+------+---------------------+------+---------+-------+------+-------+
|  1 | SIMPLE      | orders | ref  | idx_user_status_time| idx  | 10      | const |  50  | NULL  |
+----+-------------+--------+------+---------------------+------+---------+-------+------+-------+
```

### 4.2 核心字段解读

```sql
-- type（访问类型）：性能从好到差
-- system   > 系统表（只有一行）
-- const    > 主键/唯一索引等值查询（最快）
-- eq_ref   > JOIN 中用到主键/唯一索引
-- ref      > 普通索引等值查询（OK）
-- range    > 索引范围扫描（OK）
-- index    > 扫描整个索引（比全表略好）
-- ALL      > 全表扫描（🚫 必须优化）

-- key: 实际使用的索引（NULL = 没用到索引）
-- key_len: 使用的索引长度（越长 = 联合索引用的列越多）
-- rows: 预估扫描行数（越少越好）
-- Extra: 额外信息
```

### 4.3 Extra 关键信息

```sql
-- ✅ 好的信号
-- Using index           → 覆盖索引，不回表（最快）
-- Using where           → 在 server 层过滤（正常）
-- Using index condition → 索引下推（ICP，5.6+优化）

-- ⚠️ 需要关注的信号
-- Using temporary       → 用了临时表（GROUP BY / DISTINCT 未优化）
-- Using filesort        → 额外排序（ORDER BY 未用索引）
-- Using join buffer     → JOIN 没走索引（使用了连接缓冲）

-- ❌ 坏的信号
-- Using filesort + Using temporary + 大 rows → 必须优化
```

### 4.4 EXPLAIN 进阶

```sql
-- 查看真实执行（8.0.18+）
EXPLAIN ANALYZE SELECT ...
-- 显示实际耗时毫秒数 + 每步 cost

-- 查看额外信息
EXPLAIN FORMAT=JSON SELECT ...
-- JSON 格式，更详细的成本信息

-- 查看优化器考虑的索引
EXPLAIN FORMAT=TREE SELECT ...
```

---

## 5. 索引优化策略

### 5.1 联合索引列顺序

```sql
-- 原则：等值条件在前，范围条件在后
-- 区分度高的在前，低的在后

-- 需求：查询某用户某状态下的订单，按时间排序
-- ❌ 差的顺序
ALTER TABLE orders ADD INDEX idx_bad (created_at, status, user_id);
-- 范围查询 created_at 放在第一列，后面全部失效

-- ✅ 好的顺序
ALTER TABLE orders ADD INDEX idx_good (user_id, status, created_at);
-- 等值 user_id → 等值 status → 范围 + 排序 created_at

-- 实战练习：选择最优索引
-- 查询：WHERE user_id=? AND created_at BETWEEN ? AND ? ORDER BY amount
-- A: (user_id, created_at, amount)      → ❌ amount 排序失效
-- B: (user_id, amount, created_at)      → ❌ created_at 范围断了 amount
-- C: (user_id, amount)                  → ✅ 但 created_at 过滤靠 WHERE
-- 最优：如果 ORDER BY amount 是必须的，建 A，否则建 D
-- D: (user_id, created_at) + 覆盖 amount → 走 ICP
```

### 5.2 三星索引

```
⭐ 第一颗星：WHERE 条件列都在索引中（减少扫描行数）
⭐ 第二颗星：避免排序（ORDER BY 走索引顺序）
⭐ 第三颗星：覆盖索引（SELECT 的列都在索引中，不回表）

理想索引 = WHERE + ORDER BY + SELECT 的列的并集
但列太多索引会过大，通常优先前两颗星
```

### 5.3 索引下推（ICP, MySQL 5.6+）

```sql
-- 索引：(user_id, status)

-- 无 ICP（5.5-）
-- 1. 通过索引找到所有 user_id=100 的主键
-- 2. 回表取出所有匹配行
-- 3. 在 Server 层过滤 status='paid'  --- 多回了很多表

-- 有 ICP（5.6+）
-- 1. 通过索引找 user_id=100
-- 2. 在索引层直接过滤 status='paid'  --- 少回表
-- 3. 只回表拿符合条件的行

SELECT * FROM orders
WHERE user_id = 100 AND status = 'paid';
-- Extra: Using index condition ← ICP 在工作
```

### 5.4 MRR（Multi-Range Read）

```sql
-- 回表随机读 → 顺序读优化

-- 没有 MRR：按索引顺序回表 → 随机 I/O
-- idx_name: (a,1) (b,5) (c,3) → 回表页号: 1, 5, 3 乱序

-- MRR 启用：收集主键 → 排序 → 批量回表
-- 回表页号: 1, 3, 5 顺序 → 顺序 I/O

SET optimizer_switch = 'mrr=on,mrr_cost_based=off';

-- 适用于：范围查询大量回表的场景
SELECT * FROM orders WHERE created_at BETWEEN '2026-01-01' AND '2026-06-01';
```

---

## 6. 多表关联优化

### 6.1 JOIN 优化原则

```sql
-- 原则：小表驱动大表
-- 被驱动表（内层循环）的关联字段必须有索引

-- ❌ 坏查询
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
JOIN orders o ON u.id = o.user_id    -- orders 没有 user_id 索引
GROUP BY u.id;
-- 每查一个 user，全表扫 orders → O(n × m)

-- ✅ 好查询
ALTER TABLE orders ADD INDEX idx_user (user_id);

SELECT u.name, COUNT(o.id) AS order_count
FROM users u
JOIN orders o ON u.id = o.user_id    -- orders.user_id 有索引
GROUP BY u.id;
-- 每查一个 user，索引精准定位 orders → O(n × log m)

-- JOIN 类型选择（从好到坏）
-- eq_ref    → 被驱动表用主键/唯一索引（最快）
-- ref       → 被驱动表用普通索引
-- range     → 被驱动表范围扫描
-- index     → 扫描索引
-- ALL       → 全表扫描（❌）
```

### 6.2 JOIN 实战优化

```sql
-- 场景：查询每个用户的最新订单

-- ❌ 子查询方案（差）
SELECT u.*,
    (SELECT o.order_no FROM orders o
     WHERE o.user_id = u.id
     ORDER BY o.created_at DESC LIMIT 1
    ) AS latest_order
FROM users u;
-- 对每个 user 执行一次子查询 → N+1 地狱

-- ⚠️ JOIN + MAX（中等）
SELECT u.name, o.order_no, o.created_at
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN (
    SELECT user_id, MAX(created_at) AS max_time
    FROM orders
    GROUP BY user_id
) latest ON o.user_id = latest.user_id AND o.created_at = latest.max_time;

-- ✅ 窗口函数（8.0+，最优）
SELECT u.name, latest.order_no, latest.amount
FROM users u
JOIN (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) latest ON u.id = latest.user_id AND latest.rn = 1;

-- 索引配合
ALTER TABLE orders ADD INDEX idx_user_time (user_id, created_at DESC);
```

### 6.3 子查询 vs JOIN vs EXISTS

```sql
-- 场景：找有已支付订单的用户

-- ❌ IN 子查询（老版本优化差，8.0 已半连接优化）
SELECT * FROM users
WHERE id IN (
    SELECT user_id FROM orders WHERE status = 'paid'
);

-- ✅ EXISTS（找到一条即返回，适合外表大内表小）
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.status = 'paid'
);

-- ✅ JOIN（需要去重）
SELECT DISTINCT u.* FROM users u
JOIN orders o ON u.id = o.user_id AND o.status = 'paid';

-- 索引核心：orders(user_id, status)
```

---

## 7. 分页优化

### 7.1 深分页问题

```sql
-- ❌ LIMIT 深分页：要扔掉前 100万行
SELECT * FROM orders ORDER BY id LIMIT 1000000, 20;
-- 实际扫描 1000020 行，每页都要重新排 → 越来越慢
```

### 7.2 游标分页（推荐）

```sql
-- ✅ 方案1：记住上次的 id
SELECT * FROM orders
WHERE id > 1000000
ORDER BY id
LIMIT 20;
-- 仅扫描 20 行！但必须连续递增，不能跳页

-- ✅ 方案2：带索引的游标
SELECT * FROM orders
WHERE (created_at, id) > ('2026-01-01', 1000000)
ORDER BY created_at, id
LIMIT 20;
-- 适合按时间线翻页
```

### 7.3 延迟关联

```sql
-- ✅ 先通过覆盖索引取主键，再关联拿数据
SELECT o.* FROM orders o
JOIN (
    SELECT id FROM orders
    WHERE user_id = 100
    ORDER BY created_at DESC
    LIMIT 100000, 20
) tmp ON o.id = tmp.id;

-- 内层只扫索引（不回表），拿到20个id后再精准回表
-- 需要索引：idx_user_time (user_id, created_at, id)
```

### 7.4 PHP 实现

```php
final class CursorPaginator
{
    public function paginate(
        string $table,
        string $cursorField = 'id',
        ?int $after = null,
        int $limit = 20
    ): array {
        $query = DB::table($table)
            ->orderBy($cursorField);

        if ($after !== null) {
            $query->where($cursorField, '>', $after);
        }

        $rows = $query->limit($limit + 1)->get();

        $hasMore = $rows->count() > $limit;
        if ($hasMore) $rows->pop();

        return [
            'data'     => $rows,
            'has_more' => $hasMore,
            'next'     => $hasMore ? $rows->last()->$cursorField : null,
        ];
    }
}

// API 响应
// GET /orders?after=1000000&limit=20
// Response:
// { "data": [...], "has_more": true, "next_cursor": 1000025 }
```

---

## 8. 排序与分组优化

### 8.1 排序优化

```sql
-- 需求：某用户订单按金额排序
-- 索引：(user_id) 不够！排序会 filesort

-- ✅ 建索引覆盖排序字段
ALTER TABLE orders ADD INDEX idx_user_amount (user_id, amount);

SELECT * FROM orders
WHERE user_id = 100
ORDER BY amount DESC
LIMIT 10;
-- Extra: NULL（没有 filesort，完美）

-- ❌ 但如果有范围条件，排序会失效
SELECT * FROM orders
WHERE user_id = 100 AND created_at > '2026-01-01'
ORDER BY amount DESC
LIMIT 10;
-- created_at 不在索引中 → 排序用不到

-- ✅ 联合索引带上范围和排序
ALTER TABLE orders ADD INDEX idx_user_time_amount
    (user_id, created_at, amount);
-- 范围后 amount 排序列也失效
-- 只能 user_id + created_at 走索引，amount 仍然 filesort

-- 最优方案：让 ORDER BY 列紧跟等值列，范围列放最后
ALTER TABLE orders ADD INDEX idx_user_amount_time
    (user_id, amount, created_at);
-- WHERE user_id=100 ORDER BY amount → 完美
-- 但 created_at 条件只能 ICP 过滤
```

### 8.2 GROUP BY 优化

```sql
-- GROUP BY 走索引的两个条件
-- 1. GROUP BY 的列是索引的最左前缀
-- 2. 不能有 WHERE 范围条件打断

-- ✅ 能用索引
ALTER TABLE orders ADD INDEX idx_status (status, created_at);
SELECT status, COUNT(*) FROM orders GROUP BY status;

-- ✅ 松散索引扫描（直接跳到下一个分组值）
SELECT status FROM orders GROUP BY status;
-- 跳过组内所有值，直接读每个分组的第一个

-- ❌ 临时表 + filesort
SELECT user_id, COUNT(*) FROM orders
WHERE amount > 100
GROUP BY user_id
ORDER BY COUNT(*) DESC;
-- 索引不够 → 临时表 → filesort

-- ✅ 优化：应用层排序或改成覆盖索引
ALTER TABLE orders ADD INDEX idx_user_amount (user_id, amount);
```

### 8.3 Using temporary 排查

```sql
-- 多列 GROUP BY
SELECT a, b FROM t GROUP BY a, b;  -- 可能临时表
-- 优化：用联合索引
ALTER TABLE t ADD INDEX idx_a_b (a, b);

-- DISTINCT + ORDER BY 不同列
SELECT DISTINCT a FROM t ORDER BY b;
-- 需要临时表来同时满足去重和排序

-- 优化：用 GROUP BY 替代，或调整业务逻辑
SELECT a FROM t GROUP BY a ORDER BY MAX(b);
```

---

## 9. 复杂查询实战

### 9.1 树形结构（递归 CTE）

```sql
-- 商品分类树（8.0+）
WITH RECURSIVE category_tree AS (
    -- 锚点（根节点）
    SELECT id, name, parent_id, 0 AS depth, CAST(id AS CHAR(200)) AS path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    -- 递归
    SELECT c.id, c.name, c.parent_id, ct.depth + 1,
           CONCAT(ct.path, ',', c.id)
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
    WHERE ct.depth < 10  -- 深度限制防死循环
)
SELECT * FROM category_tree ORDER BY path;
```

```
结果：
id  name           parent_id  depth  path
1   电子产品        NULL       0      1
2   手机            1          1      1,2
3   苹果            2          2      1,2,3
4   安卓            2          2      1,2,4
5   电脑            1          1      1,5
```

```sql
-- 查询某节点下的所有子节点（含自身）
WITH RECURSIVE subtree AS (
    SELECT id FROM categories WHERE id = 2
    UNION ALL
    SELECT c.id FROM categories c
    JOIN subtree s ON c.parent_id = s.id
)
SELECT * FROM categories WHERE id IN (SELECT id FROM subtree);

-- 索引：parent_id 必须有索引
ALTER TABLE categories ADD INDEX idx_parent (parent_id);
```

### 9.2 行转列

```sql
-- 原始数据（订单每日统计）
-- user_id | date       | amount
-- 1       | 2026-01-01 | 100
-- 1       | 2026-01-02 | 150
-- 1       | 2026-01-03 | 200

-- 转为：user_id | day1 | day2 | day3

-- ✅ CASE WHEN 方案
SELECT user_id,
    SUM(CASE WHEN date = '2026-01-01' THEN amount ELSE 0 END) AS day1,
    SUM(CASE WHEN date = '2026-01-02' THEN amount ELSE 0 END) AS day2,
    SUM(CASE WHEN date = '2026-01-03' THEN amount ELSE 0 END) AS day3
FROM daily_orders
GROUP BY user_id;

-- ✅ IF 方案
SELECT user_id,
    SUM(IF(date = '2026-01-01', amount, 0)) AS day1,
    SUM(IF(date = '2026-01-02', amount, 0)) AS day2,
    SUM(IF(date = '2026-01-03', amount, 0)) AS day3
FROM daily_orders
GROUP BY user_id;
```

### 9.3 连续登录天数

```sql
-- 用户签到表：user_id, login_date
-- 需求：找出连续登录 ≥3 天的用户

WITH numbered AS (
    SELECT user_id, login_date,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) AS rn
    FROM user_logins
    GROUP BY user_id, login_date  -- 去重同一天多次
),
grouped AS (
    SELECT user_id, login_date, rn,
        DATE_SUB(login_date, INTERVAL rn DAY) AS grp  -- 连续日期的分组标记
    FROM numbered
)
SELECT user_id, MIN(login_date) AS start_date,
       MAX(login_date) AS end_date,
       COUNT(*) AS consecutive_days
FROM grouped
GROUP BY user_id, grp
HAVING consecutive_days >= 3
ORDER BY user_id, start_date;
```

```
原理：
日期          rn   date - rn
2026-01-01    1    2025-12-31
2026-01-02    2    2025-12-31 ← 同一组
2026-01-03    3    2025-12-31 ← 连续3天
2026-01-05    4    2026-01-01 ← 新组（断了1天）
```

### 9.4 TOP N 每组

```sql
-- 需求：每个分类下销售额最高的 3 个商品

SELECT category_id, product_id, product_name, total_sales, rn
FROM (
    SELECT
        p.category_id,
        p.id AS product_id,
        p.name AS product_name,
        SUM(o.amount) AS total_sales,
        ROW_NUMBER() OVER (PARTITION BY p.category_id
                           ORDER BY SUM(o.amount) DESC) AS rn
    FROM products p
    JOIN order_items oi ON p.id = oi.product_id
    JOIN orders o ON oi.order_id = o.id
    WHERE o.created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
    GROUP BY p.category_id, p.id, p.name
) ranked
WHERE rn <= 3;
```

### 9.5 用户行为漏斗

```sql
-- 需求：某日各转化步骤的用户数
-- 漏斗：浏览 → 加购 → 下单 → 支付

SELECT
    COUNT(DISTINCT CASE WHEN event = 'view' THEN user_id END) AS view_users,
    COUNT(DISTINCT CASE WHEN event = 'add_cart' THEN user_id END) AS cart_users,
    COUNT(DISTINCT CASE WHEN event = 'create_order' THEN user_id END) AS order_users,
    COUNT(DISTINCT CASE WHEN event = 'pay' THEN user_id END) AS pay_users
FROM user_events
WHERE created_at BETWEEN '2026-01-01 00:00:00' AND '2026-01-01 23:59:59';

-- 带转化率
SELECT
    '浏览 → 加购' AS step,
    ROUND(
        COUNT(DISTINCT CASE WHEN event = 'add_cart' THEN user_id END) * 100.0 /
        NULLIF(COUNT(DISTINCT CASE WHEN event = 'view' THEN user_id END), 0),
        2
    ) AS conversion_rate
FROM user_events
WHERE created_at BETWEEN '2026-01-01 00:00:00' AND '2026-01-01 23:59:59';
```

### 9.6 留存分析

```sql
-- 需求：新用户次日/7日/30日留存

SELECT
    register_date,
    COUNT(DISTINCT u.id) AS new_users,
    ROUND(COUNT(DISTINCT CASE
        WHEN DATEDIFF(l.login_date, u.register_date) = 1
        THEN u.id END) * 100.0 / COUNT(DISTINCT u.id), 2) AS day1_retention,
    ROUND(COUNT(DISTINCT CASE
        WHEN DATEDIFF(l.login_date, u.register_date) = 7
        THEN u.id END) * 100.0 / COUNT(DISTINCT u.id), 2) AS day7_retention,
    ROUND(COUNT(DISTINCT CASE
        WHEN DATEDIFF(l.login_date, u.register_date) = 30
        THEN u.id END) * 100.0 / COUNT(DISTINCT u.id), 2) AS day30_retention
FROM (
    SELECT id, DATE(created_at) AS register_date
    FROM users
    WHERE created_at >= '2026-01-01'
) u
LEFT JOIN user_logins l ON u.id = l.user_id
    AND l.login_date BETWEEN u.register_date
        AND DATE_ADD(u.register_date, INTERVAL 30 DAY)
GROUP BY register_date
ORDER BY register_date DESC;
```

---

## 10. 慢查询定位与优化

### 10.1 开启慢查询

```sql
-- 查看配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 配置（建议）
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;          -- 超过1秒记录
SET GLOBAL log_queries_not_using_indexes = ON;  -- 未用索引也记录
SET GLOBAL log_slow_admin_statements = ON;      -- ALTER 也记录

-- 生产建议写入 my.cnf
-- slow_query_log=1
-- slow_query_log_file=/var/log/mysql/slow.log
-- long_query_time=1
-- log_queries_not_using_indexes=1
```

### 10.2 分析慢查询

```bash
# mysqldumpslow
mysqldumpslow -s t -t 10 slow.log     # 按时间排序 Top 10
mysqldumpslow -s c -t 10 slow.log     # 按次数排序 Top 10
mysqldumpslow -s r -t 10 slow.log     # 按扫描行数排序 Top 10

# pt-query-digest（更强大）
pt-query-digest slow.log > slow_report.txt
pt-query-digest slow.log --since '2026-01-01' > report.txt
```

### 10.3 优化流程

```
发现慢查询
    │
    ├─ EXPLAIN 看执行计划
    │
    ├─ profiling 看耗时分布
    │   SET profiling = 1;
    │   SELECT ...;
    │   SHOW PROFILES;
    │   SHOW PROFILE FOR QUERY 1;
    │
    ├─ 检查索引
    │   ├─ 是否用到索引？
    │   ├─ 是否用了不合适的索引？
    │   ├─ 是否缺少联合索引？
    │   └─ 是否覆盖索引？
    │
    ├─ 检查 SQL 写法
    │   ├─ SELECT * 是否必要？
    │   ├─ 能否用 LIMIT？
    │   ├─ 子查询能否改 JOIN？
    │   └─ 是否有隐式类型转换？
    │
    └─ 检查数据量
        ├─ 是否需要分区？
        ├─ 是否需要归档？
        └─ 是否需要读写分离？
```

---

## 11. 索引失效场景全集

```sql
-- 假设索引 idx_a_b_c (a, b, c)

-- ❌ 1. 违反最左前缀
SELECT * FROM t WHERE b = 1 AND c = 2;         -- 跳过了 a
SELECT * FROM t WHERE a = 1 AND c = 2;          -- 只能用 a

-- ❌ 2. 范围查询右边失效
SELECT * FROM t WHERE a = 1 AND b > 10 AND c = 1;
--                           ✅      ⚠️      ❌

-- ❌ 3. 列上做运算（即使用到了索引，也是全索引扫描）
SELECT * FROM t WHERE a + 1 = 10;              -- 改成 a = 9
SELECT * FROM orders WHERE YEAR(created_at) = 2026;  -- 改成范围

-- ❌ 4. 对索引列使用函数
SELECT * FROM users WHERE LOWER(email) = 'khz@example.com';
-- 改为：email = 'khz@example.com'（存储时统一小写）
-- 或建函数索引（8.0.13+）
ALTER TABLE users ADD INDEX idx_email_lower ((LOWER(email)));

-- ❌ 5. 隐式类型转换（致命）
-- varchar 列与整数比较
SELECT * FROM users WHERE phone = 13800138000;
-- 实际执行：WHERE CAST(phone AS SIGNED) = 13800138000
-- 索引失效！改成：WHERE phone = '13800138000'

-- ❌ 6. LIKE 前置通配符
SELECT * FROM articles WHERE title LIKE '%redis%';
-- 索引失效！仅 'redis%' 能用索引
-- 替代方案：全文索引（见第13节）

-- ❌ 7. NOT / != / <>
SELECT * FROM t WHERE a != 1;   -- 全表扫描
SELECT * FROM t WHERE a NOT IN (1, 2, 3);  -- 全表扫描
-- 改用覆盖索引或分批处理

-- ❌ 8. OR 两边的列不同时建了索引
SELECT * FROM t WHERE a = 1 OR b = 2;
-- 需要分别建 idx_a 和 idx_b

-- ❌ 9. IS NULL / IS NOT NULL
-- 字段允许 NULL，索引中不存 NULL 值
SELECT * FROM t WHERE b IS NULL;    -- 不走索引
-- 如果确实需要查询 NULL，给默认值或新建一个可为 NULL 的索引

-- ❌ 10. ORDER BY + LIMIT 深分页（前面已讲）

-- ✅ 口诀
-- 全值匹配我最爱，最左前缀要遵守
-- 带头大哥不能死，中间兄弟不能断
-- 索引列上少计算，范围之后全失效
-- LIKE 百分写最右，覆盖索引不写 *
-- 不等空值还有 OR，索引失效要少用
```

---

## 12. 大表索引策略

### 12.1 在线 DDL

```sql
-- MySQL 8.0 大部分 ALTER TABLE 支持 ONLINE（不锁表）
-- 但仍有代价：CPU、磁盘 I/O、主从延迟

-- 查看是否支持 online
-- https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html

ALTER TABLE orders
ADD INDEX idx_user_status (user_id, status),
ALGORITHM=INPLACE, LOCK=NONE;  -- 优先 online + 无锁

-- 大表加索引：用 pt-online-schema-change
pt-online-schema-change \
    --alter "ADD INDEX idx_user_status (user_id, status)" \
    D=mydb,t=orders \
    --execute
```

### 12.2 索引选择

```sql
-- 不是索引越多越好！
-- 查询太少 → 浪费空间 + 拖慢写入
-- 查询太多 → 维护成本高

-- 查看冗余索引
SELECT * FROM sys.schema_redundant_indexes;

-- 查看从未使用的索引
SELECT * FROM sys.schema_unused_indexes;
-- ⚠️ 谨慎删除：可能只是统计周期内没用

-- 删除无用索引
ALTER TABLE orders DROP INDEX idx_never_used;
```

### 12.3 分区表

```sql
-- 按时间分区（按月）
ALTER TABLE orders
PARTITION BY RANGE (TO_DAYS(created_at)) (
    PARTITION p202601 VALUES LESS THAN (TO_DAYS('2026-02-01')),
    PARTITION p202602 VALUES LESS THAN (TO_DAYS('2026-03-01')),
    PARTITION p202603 VALUES LESS THAN (TO_DAYS('2026-04-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- 查询自动裁剪分区
SELECT * FROM orders WHERE created_at = '2026-01-15';
-- 只扫描 p202601 分区

-- 快速删除旧数据（秒级）
ALTER TABLE orders TRUNCATE PARTITION p202601;  -- 秒删
-- vs DELETE FROM orders WHERE ... （可能跑几小时）
```

### 12.4 冷热分离

```
热数据（近3个月）→ 同一张表，频繁访问
温数据（3-12个月）→ 另一张表或分区
冷数据（1年以上）→ 归档表或对象存储

访问模式差异 → 索引策略不同
热数据：大量联合索引 + 覆盖索引
冷数据：只建必要的查询索引
```

---

## 13. 主从复制与读写分离

### 13.1 主从架构

```
                   ┌─────────────┐
             ┌─────│  Master (主) │─────┐
             │     │  写操作      │     │
             │     └─────────────┘     │
             │   binlog 复制            │
             ▼                         ▼
      ┌─────────────┐         ┌─────────────┐
      │ Slave 1 (从) │         │ Slave 2 (从) │
      │  只读/报表    │         │  只读/备份   │
      └─────────────┘         └─────────────┘

数据流：Master 写 → binlog → I/O 线程拉取 → SQL 线程重放 → Slave

主从延迟来源：
- 网络传输 binlog
- Slave 单线程回放（5.6-）
- 大事务
- Slave 机器性能差
```

### 13.2 配置主从

```bash
# ═══ Master my.cnf ═══
[mysqld]
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = ROW            # 推荐 ROW（不易丢数据）
binlog_row_image = MINIMAL     # 减少 binlog 大小
expire_logs_days = 7
gtid_mode = ON                 # 8.0 推荐 GTID
enforce_gtid_consistency = ON

# ═══ Slave my.cnf ═══
[mysqld]
server-id = 2
relay_log = /var/log/mysql/relay-bin.log
gtid_mode = ON
enforce_gtid_consistency = ON
read_only = ON                 # 从库只读
log_slave_updates = ON         # 级联复制时需要

# ═══ 在 Master 创建复制用户 ═══
CREATE USER 'repl'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

# ═══ 在 Slave 执行 ═══
CHANGE MASTER TO
    MASTER_HOST = '192.168.1.100',
    MASTER_PORT = 3306,
    MASTER_USER = 'repl',
    MASTER_PASSWORD = 'password',
    MASTER_AUTO_POSITION = 1;   -- GTID 模式

START SLAVE;
SHOW SLAVE STATUS\G
-- 关注：Slave_IO_Running 和 Slave_SQL_Running 都是 Yes
-- Seconds_Behind_Master 主从延迟秒数
```

### 13.3 主从延迟处理

```sql
-- 查看延迟
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: 0 表示无延迟

-- 延迟原因排查
-- 1. 大事务（几万行 UPDATE/DELETE）
-- 2. 批量操作没分批
-- 3. 从库硬件差
-- 4. 从库同时做报表查询

-- ✅ 减小延迟
-- 并行复制（5.7+）
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL slave_parallel_workers = 4;  -- 4 个并行线程

-- 拆分大事务为小事务
-- ❌ 一次更新 100 万行
UPDATE orders SET status = 'archived' WHERE id BETWEEN 1 AND 1000000;

-- ✅ 分批执行
UPDATE orders SET status = 'archived' WHERE id BETWEEN 1 AND 10000;
UPDATE orders SET status = 'archived' WHERE id BETWEEN 10001 AND 20000;
-- ... 循环执行
```

### 13.4 PHP 读写分离

```php
/**
 * 简单的读写分离实现
 */
final class ReadWriteConnection
{
    private array $readConnections;  // 从库连接池
    private int $readIndex = 0;

    public function __construct(
        private \PDO $writeConnection,
        array $readConnectionConfigs,
    ) {
        foreach ($readConnectionConfigs as $config) {
            $this->readConnections[] = new \PDO(
                "mysql:host={$config['host']};dbname={$config['database']}",
                $config['username'],
                $config['password'],
                [
                    \PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION,
                    \PDO::ATTR_DEFAULT_FETCH_MODE => \PDO::FETCH_ASSOC,
                ]
            );
        }
    }

    /**
     * 获取读连接（轮询从库）
     */
    public function read(): \PDO
    {
        if (empty($this->readConnections)) {
            return $this->writeConnection;  // 没有从库，恢复写库读
        }

        $conn = $this->readConnections[$this->readIndex];
        $this->readIndex = ($this->readIndex + 1) % count($this->readConnections);
        return $conn;
    }

    /**
     * 获取写连接
     */
    public function write(): \PDO
    {
        return $this->writeConnection;
    }
}

// 使用
$db = new ReadWriteConnection($writePDO, [
    ['host' => 'slave1', 'database' => 'mydb', 'username' => 'root', 'password' => 'pass'],
    ['host' => 'slave2', 'database' => 'mydb', 'username' => 'root', 'password' => 'pass'],
]);

// 读操作用从库
$stmt = $db->read()->query('SELECT * FROM orders WHERE user_id = 100');

// 写操作用主库，写完立刻读需要指定用主库
$db->write()->exec("UPDATE orders SET status = 'paid' WHERE id = 123");
$order = $db->write()->query('SELECT * FROM orders WHERE id = 123');  // 主库读防延迟
```

### 13.5 Laravel 读写分离

```php
// config/database.php
'mysql' => [
    'read' => [
        ['host' => '192.168.1.101'],  // slave 1
        ['host' => '192.168.1.102'],  // slave 2
    ],
    'write' => [
        ['host' => '192.168.1.100'],  // master
    ],
    'sticky' => true,  // ⭐ 重要：同一请求周期内，写过就用主库读
    'driver' => 'mysql',
    'database' => 'mydb',
    'username' => 'root',
    'password' => 'pass',
    'charset' => 'utf8mb4',
],

// sticky = true 的行为：
// 1. 读操作 → 从库
// 2. 写操作 → 主库
// 3. 同一请求再读 → 主库（防止主从延迟导致读不到刚写的数据）
// 4. 下一个请求 → 重新从从库开始

// 手动指定连接
DB::connection('mysql::write')->select('...');  // 强制主库
DB::connection('mysql::read')->select('...');   // 强制从库
```

### 13.6 一主多从 + 延迟从库

```
架构升级：

                Master (写)
                  │
     ┌────────────┼────────────┐
     ▼            ▼            ▼
  Slave 1      Slave 2      Slave 3
  (线上读)     (线上读)     (延迟30分钟备份)

延迟从库用途：
- 误删数据恢复（有30分钟窗口回滚）
- 全量备份（不影响线上）

配置延迟：
CHANGE MASTER TO MASTER_DELAY = 1800;  -- 延迟 30 分钟

误删恢复操作：
1. 立即 STOP SLAVE（保留数据）
2. 从延迟从库导出被删数据
3. 导入主库
```

### 13.7 半同步复制（防数据丢失）

```sql
-- 异步复制：Master 提交 → 返回客户端 → 写 binlog → Slave 异步拉取
--   风险：Master 宕机，binlog 没传到 Slave → 数据丢失

-- 半同步复制：Master 提交 → 写 binlog → 等至少1个 Slave 确认收到 → 返回客户端
--   性能损失约 20-30%，但数据不丢

-- Master 安装插件
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = ON;
SET GLOBAL rpl_semi_sync_master_timeout = 10000;  -- 10秒超时降级为异步

-- Slave 安装插件
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET GLOBAL rpl_semi_sync_slave_enabled = ON;

-- 监控
SHOW STATUS LIKE 'Rpl_semi_sync%';
```

### 13.8 MHA / Orchestrator（高可用）

```
故障自动切换：

MHA（Master High Availability）：
1. 监控 Master 存活
2. Master 宕机 → 选一个数据最新的 Slave 提升为新 Master
3. 其他 Slave 指向新 Master
4. 全过程 10-30 秒

Orchestrator（更现代）：
- Web UI 可视化管理拓扑
- 自动故障检测 + 切换
- 支持中间件 ProxySQL 联动

# 简单自愈脚本思路（非 MHA 的轻量方案）
# 1. 检测：mysqladmin ping -h master
# 2. 确认：多次检测都失败
# 3. 选主：选 Seconds_Behind_Master 最小的 Slave
# 4. 切换：STOP SLAVE; RESET SLAVE ALL; 其他从库 CHANGE MASTER TO 新主
# 5. 通知：发告警
```

---

## 14. 分库分表

### 14.1 什么时候需要分库分表

```
单个 MySQL 实例的极限：
- 磁盘：单表 2000 万行开始明显变慢
- 连接数：单实例约 500-1000 并发连接
- 写入：单实例约 3000-5000 TPS

分库分表信号：
✅ 单表数据量 > 2000 万
✅ 磁盘占用 > 500GB
✅ 写入瓶颈（TPS 到顶）
✅ 数据库连接数扛不住
✅ 单表 ALTER TABLE 要跑几小时

但先尝试这些：
- 优化索引和 SQL
- 读写分离
- 缓存（Redis）
- 冷热数据分离
- 分区表
```

### 14.2 切分策略

#### 垂直分库

```
拆分前：
┌─────────────── 单库 ═══════════════┐
│ users   orders   products   payment │
│ logs    reports  settings   files   │
└────────────────────────────────────┘

拆分后：
┌──── 用户库 ────┐ ┌──── 订单库 ────┐ ┌──── 支付库 ────┐
│ users          │ │ orders         │ │ payment        │
│ user_profiles  │ │ order_items    │ │ transactions   │
│ user_logins    │ │ carts          │ │ refunds        │
└────────────────┘ └────────────────┘ └────────────────┘

特点：
- 表结构不同，按业务模块拆分
- JOIN 不能跨库（需要应用层拼装）
- 事务不能跨库（需要分布式事务或最终一致性）
```

#### 水平分表

```
拆分前：
orders 单表 1 亿行

拆分后：
orders_0: user_id % 4 = 0 的数据
orders_1: user_id % 4 = 1 的数据
orders_2: user_id % 4 = 2 的数据
orders_3: user_id % 4 = 3 的数据

特点：
- 表结构完全相同
- 单表数据量可控
- 查询需要知道分片键
```

### 14.3 分片算法

```php
/**
 * 哈希取模（最常用）
 * 优点：数据分布均匀
 * 缺点：扩容时要重新哈希（迁移成本高）
 */
final class ModSharding
{
    private int $shardCount;

    public function __construct(int $shardCount = 4)
    {
        $this->shardCount = $shardCount;
    }

    public function getShard(string $key): int
    {
        // CRC32 比 MD5 快得多
        return abs(crc32($key)) % $this->shardCount;
    }

    public function getTableName(string $table, string $key): string
    {
        return $table . '_' . $this->getShard($key);
    }
}

// 使用
$sharding = new ModSharding(4);
$table = $sharding->getTableName('orders', 'user:123');
// → orders_2

$sql = "SELECT * FROM {$table} WHERE user_id = 123";
```

```php
/**
 * 一致性哈希（减少扩容迁移量）
 * 优点：加节点只影响相邻节点
 * 缺点：可能数据不均衡（需要虚拟节点）
 */
final class ConsistentHash
{
    private array $nodes = [];
    private int $virtualNodes = 150; // 虚拟节点数

    public function addNode(string $node): void
    {
        for ($i = 0; $i < $this->virtualNodes; $i++) {
            $hash = crc32("{$node}#{$i}");
            $this->nodes[$hash] = $node;
        }
        ksort($this->nodes);
    }

    public function getNode(string $key): string
    {
        if (empty($this->nodes)) {
            throw new \RuntimeException('没有可用节点');
        }

        $hash = crc32($key);

        // 顺时针找第一个节点
        foreach ($this->nodes as $nodeHash => $node) {
            if ($hash <= $nodeHash) return $node;
        }

        // 没找到 → 回到环的开头
        return reset($this->nodes);
    }
}

// $hash->addNode('db_node_1');
// $hash->addNode('db_node_2');
// $node = $hash->getNode('user:123');
```

```php
/**
 * 按时间分片
 * 适合：日志、流水、时序数据
 * 优点：天然按时间查询裁剪，扩容简单
 * 缺点：写入热点（最新分片压力大）
 */
final class TimeSharding
{
    public function getTableName(string $base, \DateTimeInterface $time): string
    {
        return $base . '_' . $time->format('Ym');  // orders_202601
    }

    public function getTableNames(string $base, string $start, string $end): array
    {
        // 跨月查询需要 UNION ALL
        $tables = [];
        $start = new \DateTime($start);
        $end = new \DateTime($end);

        while ($start <= $end) {
            $tables[] = $base . '_' . $start->format('Ym');
            $start->modify('first day of next month');
        }

        return $tables;
    }
}

// $tables = $sharding->getTableNames('orders', '2026-01-15', '2026-03-20');
// → ['orders_202601', 'orders_202602', 'orders_202603']
// 拼接：SELECT * FROM orders_202601
//       UNION ALL SELECT * FROM orders_202602
//       UNION ALL SELECT * FROM orders_202603
```

### 14.4 分片键选择

```
分片键 = 大部分查询 WHERE 条件里都带的那个字段

例如订单系统：
  - 分片键选 user_id（因为大部分查询是「我的订单」）
  - 但偶尔要按 order_id 查 → 需要路由表或全分片查询

选分片键的原则：
1. ⭐ 最高频的查询条件
2. 数据分布均匀（避免热点分片）
3. 尽量避免跨分片查询

常见分片键：
- 用户系统 → user_id
- 订单系统 → user_id（C 端）/ merchant_id（B 端）
- 内容系统 → article_id
- 日志系统 → created_at
```

### 14.5 跨分片查询

```php
/**
 * 处理无法用分片键定位的查询
 */
final class CrossShardQuery
{
    /**
     * 按非分片键查询（如按 order_id 查）
     */
    public function findByOrderId(string $orderId): ?array
    {
        // 方案1：基因法 — order_id 里嵌入 user_id
        // order_id = 时间戳 + user_id 后4位 + 序列号
        // 从 order_id 反解出 user_id → 定位分片
        $userId = $this->extractUserIdFromOrderId($orderId);
        $shard = $this->sharding->getShard($userId);

        return DB::table("orders_{$shard}")
            ->where('order_id', $orderId)
            ->first();
    }

    /**
     * 基因法生成 ID
     */
    public function generateOrderId(int $userId): string
    {
        // 时间毫秒 (13位) + 用户基因 (4位) + 序列号 (4位)
        $ts = (string) (microtime(true) * 1000);
        $gene = str_pad($userId % 10000, 4, '0', STR_PAD_LEFT);
        $seq = str_pad(random_int(0, 9999), 4, '0', STR_PAD_LEFT);

        return $ts . $gene . $seq;
        // 例: 1683987200000_0023_0456 → 包含 user_id 后4位
    }

    /**
     * 全分片遍历（不得已时）
     */
    public function findByStatus(string $status): array
    {
        $results = [];
        for ($i = 0; $i < $this->shardCount; $i++) {
            $rows = DB::table("orders_{$i}")
                ->where('status', $status)
                ->limit(100)
                ->get();
            $results = array_merge($results, $rows);
        }
        return $results;
    }
}
```

### 14.6 分布式 ID 生成

```php
/**
 * 雪花算法（Snowflake）— 最常用的分布式 ID 方案
 *
 * 64-bit 结构：
 * ┌────┬───────────────────┬───────────┬──────────┐
 * │ 1b │    41b 毫秒时间戳   │ 10b 机器ID │ 12b 序号  │
 * └────┴───────────────────┴───────────┴──────────┘
 */
final class SnowflakeId
{
    private const EPOCH = 1700000000000;  // 2023-11-14 基准时间
    private const MACHINE_BITS = 10;
    private const SEQUENCE_BITS = 12;

    private int $machineId;
    private int $sequence = 0;
    private int $lastTimestamp = -1;

    public function __construct(int $machineId)
    {
        $maxMachineId = (1 << self::MACHINE_BITS) - 1;
        if ($machineId < 0 || $machineId > $maxMachineId) {
            throw new \InvalidArgumentException("机器ID范围：0-{$maxMachineId}");
        }
        $this->machineId = $machineId;
    }

    public function nextId(): int
    {
        $timestamp = $this->currentTimeMillis();

        if ($timestamp < $this->lastTimestamp) {
            throw new \RuntimeException('时钟回拨！');
        }

        if ($timestamp === $this->lastTimestamp) {
            $this->sequence = ($this->sequence + 1) & ((1 << self::SEQUENCE_BITS) - 1);
            if ($this->sequence === 0) {
                // 序号用完了，等下一毫秒
                $timestamp = $this->waitNextMillis($this->lastTimestamp);
            }
        } else {
            $this->sequence = 0;
        }

        $this->lastTimestamp = $timestamp;

        return (($timestamp - self::EPOCH) << (self::MACHINE_BITS + self::SEQUENCE_BITS))
            | ($this->machineId << self::SEQUENCE_BITS)
            | $this->sequence;
    }

    private function currentTimeMillis(): int
    {
        return (int) (microtime(true) * 1000);
    }

    private function waitNextMillis(int $lastTimestamp): int
    {
        $timestamp = $this->currentTimeMillis();
        while ($timestamp <= $lastTimestamp) {
            $timestamp = $this->currentTimeMillis();
        }
        return $timestamp;
    }
}

// 使用
$snowflake = new SnowflakeId(1);  // 机器ID=1
$orderId = $snowflake->nextId();  // 7062044769984513

// 也可以用 Redis 简单方案
$redis->incr('global:order_id');  // 原子自增

// 或号段模式（从 DB 批量拿号段减少 DB 压力）
// 每次从 DB 取 1000 个号，本地消耗完再取
```

### 14.7 分表后的查询策略

```php
final class ShardedQueryBuilder
{
    /**
     * 单分片查询（带了分片键）
     */
    public function query(string $table, string $shardKey, callable $builder): array
    {
        $shard = $this->sharding->getShard($shardKey);
        $tableName = "{$table}_{$shard}";

        return $builder(DB::table($tableName));
    }

    /**
     * 全分片查询（不带分片键，慎用）
     */
    public function queryAllShards(string $table, callable $builder, int $limit = 100): array
    {
        $allResults = [];

        for ($i = 0; $i < $this->shardCount; $i++) {
            $tableName = "{$table}_{$i}";
            $results = $builder(DB::table($tableName))->limit($limit)->get();
            $allResults = array_merge($allResults, $results->toArray());
        }

        // 全分片聚合需要应用层排序
        usort($allResults, fn($a, $b) => $b['created_at'] <=> $a['created_at']);

        return array_slice($allResults, 0, $limit);
    }

    /**
     * 全分片 COUNT（需要汇总）
     */
    public function count(string $table, string $field = '*'): int
    {
        $total = 0;
        for ($i = 0; $i < $this->shardCount; $i++) {
            $total += DB::table("{$table}_{$i}")->count($field);
        }
        return $total;
    }

    /**
     * 跨分片 SUM
     */
    public function sum(string $table, string $field): int
    {
        $total = 0;
        for ($i = 0; $i < $this->shardCount; $i++) {
            $total += DB::table("{$table}_{$i}")->sum($field);
        }
        return $total;
    }
}
```

### 14.8 分表后的分页难题

```php
/**
 * 分表后 OFFSET/LIMIT 失效，需要特殊处理
 */
final class ShardedPagination
{
    /**
     * 全局分页（按时间排序，每页 20 条）
     *
     * 思路：
     *   每个分片取 limit 条 → 应用层归并排序 → 取前 limit 条
     *   适合前几页
     *
     *   深分页用游标（和单表一样）
     */
    public function paginate(string $table, int $page, int $limit = 20): array
    {
        $perShard = $limit;  // 每个分片取 limit 条
        $all = [];

        for ($i = 0; $i < $this->shardCount; $i++) {
            $rows = DB::table("{$table}_{$i}")
                ->orderBy('created_at', 'desc')
                ->limit($perShard)
                ->get()
                ->toArray();
            $all = array_merge($all, $rows);
        }

        // 归并排序
        usort($all, fn($a, $b) => $b['created_at'] <=> $a['created_at']);

        // 取第 page 页
        $offset = ($page - 1) * $limit;
        return array_slice($all, $offset, $limit);
    }

    /**
     * 游标分页（推荐）
     */
    public function cursorPaginate(string $table, ?string $afterId = null, int $limit = 20): array
    {
        $all = [];

        for ($i = 0; $i < $this->shardCount; $i++) {
            $query = DB::table("{$table}_{$i}")
                ->orderBy('id', 'desc')
                ->limit($limit);

            if ($afterId !== null) {
                $query->where('id', '<', $afterId);
            }

            $rows = $query->get()->toArray();
            $all = array_merge($all, $rows);
        }

        // 归并 + 取前 N 条
        usort($all, fn($a, $b) => $b['id'] <=> $a['id']);
        $page = array_slice($all, 0, $limit);

        return [
            'data'     => $page,
            'has_more' => count($all) > $limit,
            'next'     => end($page)['id'] ?? null,
        ];
    }
}
```

### 14.9 分布式事务

```
分库后事务不能跨库，解决方案：

方案1：最终一致性（推荐）
  创建订单流程：
  1. 订单服务：写 orders 表（分库A，立即提交）
  2. 发消息「订单创建成功」到 MQ
  3. 库存服务：消费消息，扣库存（分库B）
  4. 如果扣库存失败 → 发「回滚消息」→ 订单服务取消订单

方案2：两阶段提交（不推荐，太重）
  性能差，锁时间长，XA 协议

方案3：TCC（Try-Confirm-Cancel）
  适合资金类强一致性场景

方案4：本地消息表
  同一本地事务中：写业务数据 + 写消息表
  后台定时扫消息表 → 发 MQ → 标记已发送
```

```php
// 最终一致性示例
final class CreateOrderService
{
    public function execute(CreateOrderDTO $dto): Order
    {
        // 阶段1：创建订单 + 本地消息表（同一事务）
        DB::transaction(function () use ($dto) {
            $order = Order::create([
                'user_id' => $dto->userId,
                'amount'  => $dto->amount,
                'status'  => 'pending',
            ]);

            // 本地消息表
            OrderEvent::create([
                'order_id' => $order->id,
                'event'    => 'order_created',
                'status'   => 'pending',
            ]);
        });

        return $order;
    }
}

// 定时任务：扫描未发送的本地消息，投递到 MQ
// * * * * * php artisan order:dispatch-events
```

### 14.10 扩容方案

```
4 分片 → 8 分片（哈希取模的痛点）

方法1：停服迁移（最简单）
  1. 停机
  2. 导出所有数据
  3. 按新规则（%8）重新导入
  4. 改配置 → 启动

方法2：双写过渡（不停服）
  1. 新老分片同时写
  2. 历史数据异步迁移
  3. 校验数据一致性
  4. 切读流量到新分片
  5. 停老分片写

方法3：一致性哈希（换算法）
  见 14.3 节，扩容只影响部分数据

方法4：预分片
  初期就建 64 张表，逻辑映射到 4 个库
  orders_00 ~ orders_15 在 db0
  orders_16 ~ orders_31 在 db1
  orders_32 ~ orders_47 在 db2
  orders_48 ~ orders_63 在 db3

  扩容：迁移部分表到新库即可
  orders_16 ~ orders_31 从 db1 → db4
```

### 14.11 分库分表中间件

```
常见中间件：

ShardingSphere (Apache)：
  - JDBC 层代理，不依赖额外服务
  - 支持分库分表、读写分离、分布式事务
  - Java 生态，PHP 不直接适用

ProxySQL：
  - 数据库代理层
  - 读写分离 + 查询路由
  - PHP 无感知，直接连 ProxySQL

Vitess (CNCF)：
  - YouTube 出品
  - 完整的分库分表 + 连接池 + 高可用
  - 学习成本高

自研：
  - PHP 应用层路由（如本章示例代码）
  - 适合分片规则简单的场景
  - 分片规则复杂时推荐中间件
```

---

## 15. 全文索引

### 13.1 基础用法

```sql
-- 创建全文索引（InnoDB 5.6+）
ALTER TABLE articles ADD FULLTEXT INDEX ft_title_content (title, content);

-- 自然语言模式
SELECT *, MATCH(title, content) AGAINST('redis 缓存') AS relevance
FROM articles
WHERE MATCH(title, content) AGAINST('redis 缓存')
ORDER BY relevance DESC;

-- BOOLEAN 模式（支持 + - ~ 操作符）
SELECT * FROM articles
WHERE MATCH(title, content) AGAINST('+redis -mysql' IN BOOLEAN MODE);
-- +必须包含   -不能包含   ~降低权重   *通配符   ""精确短语

-- 查询扩展模式（自动关联词）
SELECT * FROM articles
WHERE MATCH(title, content) AGAINST('database' WITH QUERY EXPANSION);
```

### 13.2 ngram 中文分词

```sql
-- 创建 ngram 解析器（5.7.6+）
ALTER TABLE articles ADD FULLTEXT INDEX ft_title_cn (title)
    WITH PARSER ngram;

-- ngram token 大小（默认 2）
SET GLOBAL ngram_token_size = 2;  -- 两个字符一组

-- 或者用专业分词插件
-- - MySQL 版 jieba 分词
-- - 或改用 Elasticsearch
```

---

## 16. 索引与锁

### 14.1 索引影响锁范围

```sql
-- InnoDB 行锁是基于索引的！

-- 场景：更新 user_id=100 的订单
-- 表结构：orders (id PK, user_id, status)

-- ❌ user_id 无索引 → 表锁！
UPDATE orders SET status = 'cancelled' WHERE user_id = 100;
-- 没有索引 → 无法精确定位行 → 锁住所有行（表锁）

-- ✅ user_id 有索引 → 行锁
ALTER TABLE orders ADD INDEX idx_user (user_id);
UPDATE orders SET status = 'cancelled' WHERE user_id = 100;
-- 有索引 → 只锁 user_id=100 的行

-- ⚠️ 间隙锁（REPEATABLE READ 隔离级别）
-- 索引 (user_id)：值有 50, 100, 150
DELETE FROM orders WHERE user_id = 100;
-- 锁住 user_id=100 的行 + 50~100 和 100~150 的间隙
-- 导致不能 INSERT user_id=75 或 user_id=125
```

### 14.2 死锁场景

```sql
-- Session A                    Session B
-- UPDATE t SET c=1 WHERE id=5;
--                              UPDATE t SET c=2 WHERE id=10;
-- UPDATE t SET c=3 WHERE id=10;
--   (等待 B 释放 id=10)
--                              UPDATE t SET c=4 WHERE id=5;
--                                (等待 A 释放 id=5)
-- 💀 死锁！

-- ✅ 避免死锁：
-- 1. 固定访问顺序（所有事务都先锁 id 小的）
-- 2. 尽量缩小事务范围
-- 3. 合理使用索引（减少锁的范围）
-- 4. 对死锁有重试机制
```

### 14.3 PHP 死锁重试

```php
final class DeadlockRetry
{
    public function execute(callable $fn, int $maxRetries = 3): mixed
    {
        for ($i = 0; $i < $maxRetries; $i++) {
            DB::beginTransaction();
            try {
                $result = $fn();
                DB::commit();
                return $result;
            } catch (\PDOException $e) {
                DB::rollBack();

                // 死锁错误码 1213
                if ($e->errorInfo[1] == 1213 && $i < $maxRetries - 1) {
                    usleep(random_int(50000, 200000)); // 随机等待
                    continue;
                }
                throw $e;
            }
        }

        throw new \RuntimeException('操作失败，已达最大重试次数');
    }
}

// 使用
$retry = new DeadlockRetry();
$retry->execute(function () {
    DB::table('orders')->where('id', 5)->update(['status' => 'paid']);
    DB::table('orders')->where('id', 10)->update(['status' => 'paid']);
});
```

---

## 附录

### A. 索引设计检查清单

```
□ 高频查询的 WHERE 列建了索引？
□ 多列查询建了联合索引（而非多个单列索引）？
□ 联合索引列顺序：等值在前，范围/排序在后？
□ JOIN 的被驱动表关联字段有索引？
□ ORDER BY / GROUP BY 能否利用索引避免 filesort？
□ 查询列能被覆盖索引覆盖？
□ 没有冗余/重复索引？
□ 大表变更用了 pt-osc 或 gh-ost？
□ 没有 SELECT *（只需的列才查）？
□ 分页用了游标或延迟关联？
```

### B. 常用监控查询

```sql
-- 查看表索引
SHOW INDEX FROM orders;

-- 索引大小
SELECT
    database_name, table_name, index_name,
    ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE database_name = 'mydb' AND stat_name = 'size'
ORDER BY size_mb DESC;

-- 表碎片
SELECT
    table_name,
    ROUND(data_length / 1024 / 1024, 2) AS data_mb,
    ROUND(index_length / 1024 / 1024, 2) AS index_mb,
    ROUND(data_free / 1024 / 1024, 2) AS free_mb
FROM information_schema.tables
WHERE table_schema = 'mydb'
ORDER BY data_length + index_length DESC;

-- 数据未使用索引的查询（sys 库需启用）
SELECT * FROM sys.statements_with_full_table_scans
ORDER BY total_latency DESC LIMIT 10;
```

### C. 记忆口诀

```
最左前缀不能忘，带头大哥不能死
中间兄弟不能断，范围之后全失效
索引列上少计算，类型转换要不得
LIKE 百分放最右，覆盖索引不写星
不等空值还有 OR，索引失效要少用
```

---

> 📝 **索引是数据库最快的路径，但建错不如不建。理解 B+Tree 才能把索引用到极致。**
