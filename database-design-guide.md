# 数据库设计规范

> 从范式到字段类型，从命名到迁移 — 写出能扛住百万行的表结构

---

## 目录

1. [设计原则](#1-设计原则)
2. [范式与反范式](#2-范式与反范式)
3. [命名规范](#3-命名规范)
4. [字段类型选择](#4-字段类型选择)
5. [索引设计基础](#5-索引设计基础)
6. [主键策略](#6-主键策略)
7. [时间与金额](#7-时间与金额)
8. [常见表设计模板](#8-常见表设计模板)
9. [迁移管理](#9-迁移管理)
10. [设计评审清单](#10-设计评审清单)

---

## 1. 设计原则

```
7 大原则：

1. 命名一致
   全库统一的命名风格，见名知义

2. 字段最小化
   用最小的类型存数据（TINYINT 不用 INT，VARCHAR 不用 TEXT）

3. NOT NULL 优先
   有默认值就别允许 NULL（索引优化、代码简洁）

4. 主键尽量自增/有序
   InnoDB 的聚簇索引依赖主键有序插入

5. 索引按需创建
   每个索引都是写操作的负担，不在大表上建无用索引

6. 垂直拆分先于水平拆分
   列多先拆表，数据量大再分库分表

7. 读写分离是标配
   主库写入、从库读取
```

---

## 2. 范式与反范式

### 2.1 三范式

```
第一范式 (1NF)：字段不可再分
  ❌ phone: "13800138000,13900139000"    (多值)
  ✅ phone_1, phone_2 分列 / 拆到 phone 表

第二范式 (2NF)：非主键字段完全依赖于主键
  ❌ 订单表: (order_id, product_id, product_name)
  → product_name 只依赖 product_id，不是整个主键
  ✅ 拆成 order + order_item + product

第三范式 (3NF)：非主键字段不依赖其他非主键字段
  ❌ 用户表: (user_id, city_id, city_name)
  → city_name 依赖 city_id（传递依赖）
  ✅ 拆成 user + city
```

### 2.2 何时反范式

```sql
-- ═══ 冗余示例 ═══

-- 范式：每次查订单要 JOIN 用户表
SELECT o.*, u.name, u.phone
FROM orders o JOIN users u ON o.user_id = u.id
WHERE o.id = 1001;

-- 反范式：订单表冗余 user_name, user_phone
-- 好处：单表查询，速度快
-- 代价：用户改名要同步更新（可以异步 MQ 做）

CREATE TABLE orders (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    user_id BIGINT UNSIGNED NOT NULL,
    user_name VARCHAR(50) NOT NULL DEFAULT '',   -- 冗余
    user_phone VARCHAR(20) NOT NULL DEFAULT '',  -- 冗余
    -- ...
    PRIMARY KEY (id)
) ENGINE=InnoDB;

-- ═══ 何时反范式 ═══
-- 1. 查询频繁但 JOIN 代价大
-- 2. 冗余字段很少变化（如历史订单的用户名）
-- 3. 报表/统计类查询
-- 4. 需要高性能读取的场景
```

---

## 3. 命名规范

```sql
-- ═══ 命名风格：全小写 + 下划线 ═══

-- 库名
CREATE DATABASE my_project;          -- 项目名简写
CREATE DATABASE my_project_log;      -- 日志库

-- 表名（复数、业务前缀）
users           -- 用户
user_profiles   -- 用户资料
order_items     -- 订单明细
payments        -- 支付记录
logs_api_access -- 日志表用 logs_ 前缀

-- 关联表
user_roles      -- 用户-角色多对多
product_tags    -- 商品-标签

-- 字段名
id              -- 主键
user_id         -- 外键
created_at      -- 创建时间
updated_at      -- 修改时间
deleted_at      -- 软删除
status          -- 状态
type            -- 类型

-- 索引名
-- 普通索引: idx_<字段1>_<字段2>
-- 唯一索引: uk_<字段1>_<字段2>
```

---

## 4. 字段类型选择

### 4.1 数值

```sql
-- ═══ 整数 ═══
TINYINT UNSIGNED    -- 0~255 (状态/类型/布尔)
SMALLINT UNSIGNED   -- 0~65535 (分类ID/小计数)
MEDIUMINT UNSIGNED  -- 0~1600万 (较少用)
INT UNSIGNED        -- 0~42亿 (主键/ID 默认)
BIGINT UNSIGNED     -- 0~1844京 (大数据场景)

-- 选型：
status TINYINT UNSIGNED NOT NULL DEFAULT 0   -- 状态
rank TINYINT UNSIGNED NOT NULL DEFAULT 0     -- 排序
view_count INT UNSIGNED NOT NULL DEFAULT 0    -- 浏览量
user_id BIGINT UNSIGNED NOT NULL              -- 用户ID

-- ⚠️ 为什么管 unsigned？
-- INT signed: -21亿 ~ 21亿
-- INT unsigned: 0 ~ 42亿
-- 主键/计数/金额都要 unsigned

-- ═══ 小数 ═══
DECIMAL(10,2)   -- 金额（精确，推荐）
FLOAT           -- 不精确！不要存钱
DOUBLE          -- 科学计算用

-- ⚠️ 金额永远用 DECIMAL！
-- FLOAT 0.1 + 0.2 = 0.30000000000000004
```

### 4.2 字符串

```sql
-- ═══ 定长 vs 变长 ═══
CHAR(32)         -- 固定 32 字符（MD5/UUID/手机号）
VARCHAR(N)       -- 变长（N 是字符数，不是字节数）

-- 选型：
mobile CHAR(11) NOT NULL DEFAULT ''         -- 手机号
id_card CHAR(18) NOT NULL DEFAULT ''        -- 身份证
password_hash CHAR(60) NOT NULL DEFAULT ''  -- bcrypt 固定60字
token CHAR(64) NOT NULL DEFAULT ''          -- 固定长度

name VARCHAR(50) NOT NULL DEFAULT ''        -- 姓名
email VARCHAR(100) NOT NULL DEFAULT ''      -- 邮箱
title VARCHAR(200) NOT NULL DEFAULT ''      -- 标题
url VARCHAR(500) NOT NULL DEFAULT ''        -- URL

-- ═══ 长文本 ═══
TEXT             -- 64KB
MEDIUMTEXT       -- 16MB
LONGTEXT         -- 4GB

-- 大文本单独拆表（避免扫描主表）
CREATE TABLE article_contents (
    article_id BIGINT UNSIGNED NOT NULL,
    content MEDIUMTEXT NOT NULL,
    PRIMARY KEY (article_id)
);
```

### 4.3 时间与枚举

```sql
-- ═══ 时间 ═══
DATE            -- 日期 (2026-05-14)
DATETIME        -- 日期时间 (2026-05-14 15:30:00)
TIMESTAMP       -- Unix 时间戳 (1970-2038, 自动时区)
TIME            -- 时间 (15:30:00)
YEAR            -- 年份

-- 选型：
created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
birthday DATE NOT NULL DEFAULT '1970-01-01'
expire_time DATETIME NOT NULL DEFAULT '2099-12-31 23:59:59'

-- ═══ 枚举替代 ═══
-- ❌ 不推荐 ENUM（扩展要 ALTER TABLE）
-- status ENUM('active','inactive','banned')

-- ✅ 用 TINYINT + 代码注释
-- status: 0-禁用 1-启用 2-删除
status TINYINT UNSIGNED NOT NULL DEFAULT 1

-- PHP 中映射
class UserStatus {
    const ACTIVE = 1;
    const INACTIVE = 0;
    const BANNED = 2;
}

-- ═══ JSON ═══
-- 5.7.8+ 支持 JSON 类型
metadata JSON NULL
-- 适合：不常查询的扩展字段
-- 适合：结构不固定的配置
-- 不适合：需要 WHERE/索引的字段
```

---

## 5. 索引设计基础

```sql
-- ═══ 必建索引的场景 ═══
-- 1. WHERE 条件
CREATE INDEX idx_status ON orders(status);

-- 2. JOIN 关联
CREATE INDEX idx_user_id ON orders(user_id);

-- 3. ORDER BY
CREATE INDEX idx_created_at ON orders(created_at);

-- 4. 联合索引（左前缀原则）
CREATE INDEX idx_user_status_time ON orders(user_id, status, created_at);
-- 命中：WHERE user_id = 1
-- 命中：WHERE user_id = 1 AND status = 1
-- 命中：WHERE user_id = 1 AND status = 1 ORDER BY created_at
-- 不命中：WHERE status = 1  （跳过了 user_id）

-- ═══ 不应该建索引的场景 ═══
-- 1. 区分度低的字段（性别、布尔）
-- 2. 频繁更新的字段（count、score）
-- 3. 长字符串字段（不建完整索引，用前缀）
-- 4. 小表（几千行以下）
```

---

## 6. 主键策略

```sql
-- ═══ 方案对比 ═══

-- 1. 自增 ID（InnoDB 最佳）
id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
PRIMARY KEY (id)
-- 优点：有序插入、聚簇索引友好、简单
-- 缺点：暴露数据规模、分库分表困难

-- 2. UUID（分布式）
id CHAR(36) NOT NULL,
PRIMARY KEY (id)
-- 优点：全局唯一、不暴露规模
-- 缺点：无序插入（页分裂）、36字节太大

-- 3. 雪花 ID（推荐！分布式 + 大致有序）
id BIGINT UNSIGNED NOT NULL,
PRIMARY KEY (id)
-- 优点：全局唯一、基本有序、64位整数
-- PHP: hyqo/snowflake 或手动位运算

-- 4. ULID（UUID 替代）
id CHAR(26) NOT NULL,
PRIMARY KEY (id)
-- 优点：按时间排序、URL 安全、26字符
-- 缺点：InnoDB 聚簇索引没有整数快

-- ═══ 推荐 ═══
-- 单库：自增 ID
-- 分库分表/微服务：雪花 ID
-- 对外暴露：雪花 ID 或 ULID（对外用字符串ID映射自增ID）
```

```php
// 雪花 ID 生成（简化版）
final class Snowflake
{
    private static int $sequence = 0;
    private static int $lastTimestamp = 0;
    private const EPOCH = 1700000000000;  // 2023-11-15 作为纪元
    private const WORKER_ID = 1;
    private const SEQUENCE_BITS = 12;

    public static function generate(): int
    {
        $now = (int)(microtime(true) * 1000) - self::EPOCH;

        if ($now === self::$lastTimestamp) {
            self::$sequence = (self::$sequence + 1) & ((1 << self::SEQUENCE_BITS) - 1);
            if (self::$sequence === 0) {
                while ($now <= self::$lastTimestamp) {
                    $now = (int)(microtime(true) * 1000) - self::EPOCH;
                }
            }
        } else {
            self::$sequence = 0;
        }
        self::$lastTimestamp = $now;

        return ($now << 22) | (self::WORKER_ID << 12) | self::$sequence;
    }
}
```

---

## 7. 时间与金额

```sql
-- ═══ 时间统一用 DATETIME ═══
-- ✅ DATETIME: 范围 1000~9999，存储 8 字节（5.6.4+），显式时区
-- ⚠️ TIMESTAMP: 范围 1970~2038，自动转时区

-- ═══ 金额统一用 DECIMAL ═══
amount DECIMAL(12,2) UNSIGNED NOT NULL DEFAULT 0.00
-- 12位总长度，2位小数 → 最大 9999999999.99

-- ═══ NULL 处理 ═══
-- 有默认值就别用 NULL
status TINYINT UNSIGNED NOT NULL DEFAULT 0
name VARCHAR(50) NOT NULL DEFAULT ''
created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP

-- NULL 的坑：
-- SELECT * FROM users WHERE name != '张三'  → 不包括 NULL 行
-- COUNT(name) 和 COUNT(*) 结果不同
-- 索引中包含 NULL 值
```

---

## 8. 常见表设计模板

### 8.1 用户表

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL DEFAULT '',
    email VARCHAR(100) NOT NULL DEFAULT '',
    phone CHAR(11) NOT NULL DEFAULT '',
    password_hash CHAR(60) NOT NULL DEFAULT '',
    avatar VARCHAR(200) NOT NULL DEFAULT '',
    status TINYINT UNSIGNED NOT NULL DEFAULT 1 COMMENT '1-正常 0-禁用',
    login_at DATETIME NOT NULL DEFAULT '1970-01-01 00:00:00',
    login_ip VARCHAR(45) NOT NULL DEFAULT '',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uk_email (email),
    UNIQUE KEY uk_phone (phone),
    KEY idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 8.2 订单表

```sql
CREATE TABLE orders (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    order_no CHAR(20) NOT NULL DEFAULT '' COMMENT '订单号',
    user_id BIGINT UNSIGNED NOT NULL,
    total_amount DECIMAL(12,2) UNSIGNED NOT NULL DEFAULT 0.00,
    paid_amount DECIMAL(12,2) UNSIGNED NOT NULL DEFAULT 0.00,
    discount_amount DECIMAL(12,2) UNSIGNED NOT NULL DEFAULT 0.00,
    status TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '0-待支付 1-已支付 2-已发货 3-已完成 4-已取消',
    pay_type TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '0-未支付 1-微信 2-支付宝',
    pay_at DATETIME NOT NULL DEFAULT '1970-01-01 00:00:00',
    remark VARCHAR(500) NOT NULL DEFAULT '',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uk_order_no (order_no),
    KEY idx_user_id (user_id),
    KEY idx_status_created (status, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 8.3 日志/统计表

```sql
-- ═══ 日志表特点：只增不删不改，查询按时间 ═══
CREATE TABLE logs_api_access (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    user_id BIGINT UNSIGNED NOT NULL DEFAULT 0,
    method VARCHAR(10) NOT NULL DEFAULT '',
    uri VARCHAR(200) NOT NULL DEFAULT '',
    status SMALLINT UNSIGNED NOT NULL DEFAULT 0,
    duration_ms INT UNSIGNED NOT NULL DEFAULT 0,
    ip VARCHAR(45) NOT NULL DEFAULT '',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    KEY idx_created_at (created_at),
    KEY idx_user_id (user_id),
    KEY idx_uri (uri(50))  -- 前缀索引
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
-- 注：日志表可以按月分区
-- PARTITION BY RANGE (TO_DAYS(created_at)) (...)
```

### 8.4 配置/字典表

```sql
-- ═══ 配置表：一次加载常驻缓存 ═══
CREATE TABLE configs (
    id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `key` VARCHAR(50) NOT NULL DEFAULT '',
    `value` TEXT NOT NULL,
    `type` VARCHAR(20) NOT NULL DEFAULT 'string',
    remark VARCHAR(200) NOT NULL DEFAULT '',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uk_key (`key`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ═══ 字典/枚举表 ═══
CREATE TABLE dictionaries (
    id INT UNSIGNED NOT NULL AUTO_INCREMENT,
    `group` VARCHAR(30) NOT NULL DEFAULT '' COMMENT '分组',
    code VARCHAR(30) NOT NULL DEFAULT '',
    name VARCHAR(50) NOT NULL DEFAULT '',
    sort TINYINT UNSIGNED NOT NULL DEFAULT 0,
    PRIMARY KEY (id),
    KEY idx_group (code)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 9. 迁移管理

```bash
# ═══ Laravel 迁移 ═══
php artisan make:migration create_products_table
php artisan migrate
php artisan migrate:rollback
php artisan migrate:status

# ═══ Phinx（独立迁移工具） ═══
# composer require robmorgan/phinx
php vendor/bin/phinx create CreateProductsTable
php vendor/bin/phinx migrate
php vendor/bin/phinx rollback
```

```php
// ═══ 迁移文件示例 ═══
// database/migrations/2026_05_14_000001_create_products_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('products', function (Blueprint $table) {
            $table->id();                                   // BIGINT UNSIGNED AUTO_INCREMENT
            $table->string('name', 100);
            $table->decimal('price', 12, 2)->unsigned()->default(0);
            $table->integer('stock')->unsigned()->default(0);
            $table->tinyInteger('status')->unsigned()->default(1);
            $table->timestamps();                           // created_at + updated_at
            $table->softDeletes();                          // deleted_at

            $table->index('status');
            $table->index('created_at');
        });

        // 表注释
        DB::statement("ALTER TABLE products COMMENT '商品表'");
    }

    public function down(): void
    {
        Schema::dropIfExists('products');
    }
};
```

---

## 10. 设计评审清单

```
上线前逐项检查：

□ 字段类型最小化了吗？
  □ 状态用 TINYINT 不用 INT
  □ 金额用 DECIMAL 不用 FLOAT
  □ 手机号用 CHAR(11) 不用 VARCHAR

□ NULL 处理了吗？
  □ 所有字段都有合理的默认值
  □ 关键字段 NOT NULL

□ 索引合理吗？
  □ WHERE/JOIN/ORDER BY 字段有索引
  □ 联合索引左前缀正确
  □ 没有冗余索引（idx_a 和 idx_a_b 只留后者）

□ 命名一致吗？
  □ 全小写+下划线
  □ 创建/更新时间是 created_at / updated_at
  □ 主键是 id

□ 字符集正确吗？
  □ utf8mb4 + utf8mb4_unicode_ci
  □ 表级别、库级别、连接级别都一致

□ 引擎正确吗？
  □ 核心表用 InnoDB
  □ 日志/归档用 InnoDB 或 Archive

□ 有注释吗？
  □ 表有 COMMENT
  □ 状态/类型字段有 COMMENT 说明枚举值

□ 大表考虑了吗？
  □ 归档策略
  □ 分区策略
  □ 读写分离
```

---

> 🗃️ **好的数据库设计像好的地基 — 看不见但决定一切。前期半小时的设计能省下后期一个月的重构。**
