# Redis 操作说明与复杂应用

> 面向 PHP 开发者，偏实战

---

## 目录

1. [基础认知](#1-基础认知)
2. [五大数据类型](#2-五大数据类型)
3. [三种特殊类型](#3-三种特殊类型)
4. [发布订阅](#4-发布订阅)
5. [事务与 Lua](#5-事务与-lua)
6. [管道 Pipeline](#6-管道-pipeline)
7. [持久化](#7-持久化)
8. [缓存经典问题](#8-缓存经典问题)
9. [分布式锁](#9-分布式锁)
10. [延迟队列](#10-延迟队列)
11. [排行榜](#11-排行榜)
12. [限流器](#12-限流器)
13. [布隆过滤器](#13-布隆过滤器)
14. [会话与Token管理](#14-会话与token管理)
15. [计数器与统计](#15-计数器与统计)
16. [消息队列替代方案](#16-消息队列替代方案)
17. [运维与监控](#17-运维与监控)

---

## 1. 基础认知

### Redis 是什么

```
一个运行在内存中的键值数据库，也支持持久化到磁盘。
单线程事件循环模型，但 6.0+ 支持多线程网络 I/O。

核心优势：
- 极快：微秒级响应
- 丰富的数据结构
- 原子操作
- 过期机制
```

### PHP 连接

```bash
# 推荐 phpredis 扩展（性能好）
pecl install redis

# 或 composer 包
composer require predis/predis
```

```php
// phpredis 扩展风格
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);
$redis->auth('password');        // 如果有密码
$redis->select(0);               // 选择数据库 0

// 建议封装连接池或单例
final class RedisFactory
{
    private static ?Redis $instance = null;

    public static function get(): Redis
    {
        if (self::$instance === null) {
            self::$instance = new Redis();
            self::$instance->connect('127.0.0.1', 6379, 1.0);
            self::$instance->setOption(Redis::OPT_SERIALIZER, Redis::SERIALIZER_PHP);
            self::$instance->setOption(Redis::OPT_PREFIX, 'app:');
        }
        return self::$instance;
    }
}
```

---

## 2. 五大数据类型

### 2.1 String（字符串）

最基础，可存字符串/数字/序列化的对象。

```php
// 基础操作
$redis->set('user:1:name', 'khz');
$redis->get('user:1:name');              // "khz"
$redis->exists('user:1:name');           // true
$redis->del('user:1:name');

// 带过期时间
$redis->setex('sms:13800138000', 300, '1234');  // 5分钟过期
$redis->ttl('sms:13800138000');                 // 剩余秒数

// 仅当 key 不存在时设置
$redis->setnx('lock:order:123', 1);  // true=设置成功, false=已存在

// 累加/累减（原子操作）
$redis->incr('article:100:views');      // +1 返回新值
$redis->incrBy('user:5:points', 10);    // +10
$redis->decr('stock:sku:xxx');          // -1
$redis->incrByFloat('balance:1', 9.99); // 浮点

// 追加
$redis->append('log:20260513', 'error:xxx|');

// 获取子串
$redis->set('hello', 'Hello World');
$redis->getRange('hello', 0, 4);  // "Hello"

// 批量操作
$redis->mSet(['a' => 1, 'b' => 2]);
$redis->mGet(['a', 'b']);  // [1, 2]

// GETSET 先取值再设新值
$old = $redis->getSet('counter', 0);  // 拿了旧值，同时设为0
```

### 2.2 Hash（哈希）

存储对象属性，比 String 存 JSON 更省内存且可单独操作字段。

```php
// 设置
$redis->hSet('user:1', 'name', 'khz');
$redis->hSet('user:1', 'email', 'khz@example.com');
// 批量
$redis->hMSet('user:1', [
    'name'  => 'khz',
    'email' => 'khz@example.com',
    'phone' => '13800138000',
]);

// 获取
$redis->hGet('user:1', 'name');        // "khz"
$redis->hGetAll('user:1');             // 所有字段（字段多时慎用）
$redis->hMGet('user:1', ['name', 'email']);  // 指定字段

// 检查
$redis->hExists('user:1', 'email');    // true
$redis->hLen('user:1');                // 字段数量
$redis->hKeys('user:1');               // 所有字段名
$redis->hVals('user:1');               // 所有字段值

// 原子递增
$redis->hIncrBy('user:1', 'login_count', 1);
$redis->hIncrByFloat('order:123', 'total', 9.99);

// 删除
$redis->hDel('user:1', 'phone');
```

### 2.3 List（列表）

双向链表，头尾操作 O(1)，适合队列/栈/时间线。

```php
// 左右压入
$redis->lPush('queue:email', 'mail-1');      // 左进 → [mail-1]
$redis->lPush('queue:email', 'mail-2');      // → [mail-2, mail-1]
$redis->rPush('queue:email', 'mail-3');      // 右进 → [mail-2, mail-1, mail-3]

// 阻塞弹出（常用于消息队列）
$redis->blPop('queue:email', 10);   // 等10秒，无消息返回null
$redis->brPop('queue:email', 0);    // 永久等待

// 裁剪保留长度（只留最近 N 条）
$redis->lTrim('recent:orders', 0, 99);  // 只保留前100条

// 范围
$redis->lRange('queue:email', 0, -1);   // 全部
$redis->lIndex('queue:email', 0);       // 第0个元素
$redis->lLen('queue:email');            // 列表长度

// 修改 / 删除
$redis->lSet('queue:email', 0, 'new-mail');
$redis->lRem('queue:email', 'mail-1', 1);  // 删除1个匹配元素
```

### 2.4 Set（集合）

无序去重，O(1) 判存，支持交/并/差集。

```php
// 增删查
$redis->sAdd('tags:article:1', 'php', 'redis', 'cache');
$redis->sRem('tags:article:1', 'cache');
$redis->sIsMember('tags:article:1', 'php');  // true
$redis->sMembers('tags:article:1');           // 所有成员
$redis->sCard('tags:article:1');              // 成员数
$redis->sRandMember('tags:article:1', 2);     // 随机2个

// 集合运算
$redis->sAdd('user:1:friends', 2, 3, 4, 5);
$redis->sAdd('user:2:friends', 4, 5, 6, 7);

$redis->sInter('user:1:friends', 'user:2:friends');    // 共同好友 [4,5]
$redis->sUnion('user:1:friends', 'user:2:friends');    // 所有好友 [2,3,4,5,6,7]
$redis->sDiff('user:1:friends', 'user:2:friends');     // 差集 [2,3]

// 运算结果存入新集合（避免传输大量数据）
$redis->sInterStore('common_friends', 'user:1:friends', 'user:2:friends');
$redis->sMembers('common_friends');
```

### 2.5 Sorted Set（有序集合）

带分数的去重集合，根据 score 排序。排行榜利器。

```php
// 添加
$redis->zAdd('leaderboard', 9500, 'player:A');
$redis->zAdd('leaderboard', 8200, 'player:B');
$redis->zAdd('leaderboard', 7600, 'player:C');

// 修改分数（增加）
$redis->zIncrBy('leaderboard', 100, 'player:A');  // 9600

// 排名（从小到大 0-based）
$redis->zRank('leaderboard', 'player:A');    // 排第几（分最低=0）
$redis->zRevRank('leaderboard', 'player:A');  // 从高到低的排名

// 区间
$redis->zRange('leaderboard', 0, 2);              // 分数最低的3个
$redis->zRevRange('leaderboard', 0, 2, true);     // 分数最高的3个（带分数）

// 按分数区间
$redis->zRangeByScore('leaderboard', 8000, 10000);
$redis->zRevRangeByScore('leaderboard', '+inf', 8000, ['limit' => [0, 10]]);

// 成员数
$redis->zCard('leaderboard');
$redis->zCount('leaderboard', 8000, 10000);  // 分数区间内的数量

// 删除
$redis->zRem('leaderboard', 'player:C');
$redis->zRemRangeByRank('leaderboard', 0, 9);   // 删除排名最低的10个
$redis->zRemRangeByScore('leaderboard', 0, 1000);
```

---

## 3. 三种特殊类型

### 3.1 Bitmap（位图）

用 bit 做统计，极度节省内存。

```php
// 用户签到（1月1号签到了）
$redis->setBit('sign:user:1:202605', 1, 1);   // 第2天签到
$redis->setBit('sign:user:1:202605', 5, 1);   // 第6天签到

// 检查某天是否签到
$redis->getBit('sign:user:1:202605', 5);  // 1

// 签到天数
$redis->bitCount('sign:user:1:202605');

// 连续签到天数（获取整个位图后自己算）
$bitmap = $redis->get('sign:user:1:202605');

// 统计在线用户（10万用户只用 12KB 内存）
// 用户ID为偏移量
$redis->setBit('online:2026-05-13', 1001, 1);
$redis->bitCount('online:2026-05-13');   // 在线人数
```

### 3.2 HyperLogLog

统计 UV（独立访问量），12KB 内存可统计 2^64 个值，误差约 0.81%。

```php
$redis->pfAdd('uv:article:100', 'user:1', 'user:2', 'user:3');
$redis->pfAdd('uv:article:100', 'user:1');          // 重复不计
$redis->pfCount('uv:article:100');                  // 3

// 合并多页面的 UV
$redis->pfMerge('uv:total', 'uv:article:100', 'uv:article:200');
```

### 3.3 Geo（地理位置）

```php
// 添加坐标
$redis->geoAdd('shops', 121.47, 31.23, 'shop:1');   // 上海人民广场
$redis->geoAdd('shops', 121.49, 31.25, 'shop:2');
$redis->geoAdd('shops', 121.45, 31.20, 'shop:3');

// 获取坐标
$redis->geoPos('shops', 'shop:1');   // [121.47, 31.23]

// 两点间距离
$redis->geoDist('shops', 'shop:1', 'shop:2', 'km');  // 约2.8km

// 附近的人（半径2km内）
$nearby = $redis->geoRadius('shops', 121.47, 31.23, 2, 'km', [
    'sort' => 'asc',    // 按距离排序
    'withdist' => true,
    'count' => 10,
]);

// 按成员查找附近（查shop:1附近2km）
$redis->geoRadiusByMember('shops', 'shop:1', 2, 'km');
```

---

## 4. 发布订阅

```php
// ═══ 发送端 ═══
$redis->publish('channel:order', json_encode([
    'event' => 'order.paid',
    'order_id' => 123,
]));

// ═══ 接收端 ═══
$redis->subscribe(['channel:order'], function ($redis, $channel, $message) {
    $data = json_decode($message, true);
    match ($data['event']) {
        'order.paid' => handleOrderPaid($data),
        default => null,
    };
});

// 模式匹配频道
$redis->pSubscribe(['channel:*'], function ($redis, $pattern, $channel, $msg) {
    // 匹配所有 channel: 开头的消息
});
```

> ⚠️ 注意：Pub/Sub 消息不持久化，离线则丢失。生产环境建议用专业的 MQ。

---

## 5. 事务与 Lua

### 5.1 事务（MULTI/EXEC）

```php
// 注意：Redis 事务不支持回滚，只会串行执行
$redis->multi();
$redis->set('key1', 'val1');
$redis->incr('counter');
$redis->zAdd('leaderboard', 100, 'player');
$result = $redis->exec();  // 返回每步的结果数组

// 语法错误会全部丢弃，运行时错误会继续执行
$redis->multi();
$redis->set('key1', 'val1');
$redis->wrongCommand();    // 这个会失败，但不影响前面的
$redis->exec();

// 乐观锁 + 事务
$redis->watch('stock:sku:1001');   // 监视 key
$stock = $redis->get('stock:sku:1001');
if ($stock > 0) {
    $redis->multi();
    $redis->decr('stock:sku:1001');
    $result = $redis->exec();       // 如果 stock 被改了，exec 返回 null
    if ($result === false) {
        // 重试...
    }
}
$redis->unwatch();
```

### 5.2 Lua 脚本（推荐）

比事务更强大：原子执行、逻辑丰富、减少网络往返。

```lua
-- 扣减库存 Lua 脚本（原子操作）
local key = KEYS[1]
local amount = tonumber(ARGV[1])

local stock = tonumber(redis.call('get', key) or 0)
if stock < amount then
    return -1  -- 库存不足
end

redis.call('decrby', key, amount)
return stock - amount  -- 返回剩余库存
```

```php
// PHP 中调用
$script = <<<'LUA'
local key = KEYS[1]
local amount = tonumber(ARGV[1])
local stock = tonumber(redis.call('get', key) or 0)
if stock < amount then
    return -1
end
redis.call('decrby', key, amount)
return stock - amount
LUA;

// 预加载脚本（减少网络传输）
$sha = $redis->script('load', $script);

// 调用
$remaining = $redis->evalSha($sha, ['stock:sku:1001', 2], 1);
if ($remaining === -1) {
    throw new InsufficientStockException('库存不足');
}
```

更复杂的 Lua 示例 — 令牌桶限流：

```lua
-- 令牌桶限流
-- KEYS[1]: 令牌桶 key
-- ARGV[1]: 容量
-- ARGV[2]: 生成速率（每秒）
-- ARGV[3]: 当前时间（毫秒）
-- ARGV[4]: 请求的令牌数

local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

-- 获取上次状态
local last_tokens = tonumber(redis.call('hget', key, 'tokens') or capacity)
local last_time = tonumber(redis.call('hget', key, 'last_time') or now)

-- 计算新令牌
local elapsed = math.max(0, now - last_time)
local new_tokens = math.min(capacity, last_tokens + (elapsed / 1000 * rate))

-- 判断
if new_tokens < requested then
    redis.call('hset', key, 'tokens', new_tokens, 'last_time', now)
    return 0  -- 限流
end

-- 扣除令牌
redis.call('hset', key, 'tokens', new_tokens - requested, 'last_time', now)
return 1  -- 放行
```

---

## 6. 管道 Pipeline

批量发送命令，减少 RTT（往返时间）。

```php
// 不使用管道：N 次命令 = N × RTT
foreach ($users as $user) {
    $redis->hSet("user:{$user->id}", 'status', 'online');
}
// 1000个用户 ≈ 1000 × 0.5ms = 500ms

// 使用管道：N 次命令 = 1 × RTT
$pipe = $redis->multi(Redis::PIPELINE);
foreach ($users as $user) {
    $pipe->hSet("user:{$user->id}", 'status', 'online');
}
$pipe->exec();
// 1000个用户 ≈ 1 × 0.5ms = 0.5ms
```

---

## 7. 持久化

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **RDB** | 定期快照到磁盘 | 恢复快、文件小 | 可能丢最后几分钟数据 |
| **AOF** | 每条写命令追加日志 | 更安全，最多丢1秒 | 文件大、恢复慢 |
| **混合** | RDB + AOF 增量 | 兼顾恢复速度和安全性 | — |

```
# 推荐生产配置
save 900 1          # 900秒内 ≥1 次修改则快照
save 300 10
save 60 10000

appendonly yes      # 开启 AOF
appendfsync everysec  # 每秒同步一次（折中方案）
```

---

## 8. 缓存经典问题

### 8.1 缓存穿透

**现象：** 大量请求查询不存在的数据，缓存没有，全部打到数据库。

```php
// ✅ 方案1：缓存空值
final class UserCache
{
    public function getById(int $id): ?array
    {
        $key = "user:{$id}";
        $cached = $this->redis->get($key);

        if ($cached !== false) {
            return $cached ?: null;  // 空串代表缓存了"不存在"
        }

        $user = $this->db->find($id);
        $this->redis->setex($key, $user ? 3600 : 60, $user ?: '');

        return $user;
    }
}

// ✅ 方案2：布隆过滤器预判（见后文第13节）
```

### 8.2 缓存击穿

**现象：** 一个热点 key 过期瞬间，大量并发请求打到数据库。

```php
// ✅ 互斥锁方案
public function getHotData(string $key): array
{
    $data = $this->redis->get($key);
    if ($data !== false) {
        return $data;
    }

    // 抢锁重建缓存
    $lockKey = "mutex:{$key}";
    if ($this->redis->set($lockKey, 1, ['nx', 'ex' => 10])) {
        try {
            $data = $this->rebuildFromDB($key);
            $this->redis->setex($key, 3600, $data);
            return $data;
        } finally {
            $this->redis->del($lockKey);
        }
    }

    // 没抢到锁，自旋等待或返回降级数据
    for ($i = 0; $i < 10; $i++) {
        usleep(50000);  // 50ms
        $data = $this->redis->get($key);
        if ($data !== false) return $data;
    }

    return $this->getFallbackData($key);
}

// ✅ 方案2：逻辑过期（不设物理过期时间）
// key 永不过期，值里存逻辑过期时间，后台异步刷新
```

### 8.3 缓存雪崩

**现象：** 大量 key 同时过期，或者 Redis 宕机，流量全压到 DB。

```php
// ✅ 过期时间加随机偏移
$ttl = 3600 + random_int(0, 600);  // 1小时 ± 10分钟
$this->redis->setex($key, $ttl, $data);

// ✅ 多级缓存
// 本地缓存 (Caffeine/APCu) → Redis → DB

// ✅ 熔断降级
final class RedisCircuitBreaker
{
    private int $failureCount = 0;
    private int $lastFailureTime = 0;

    public function call(callable $fn): mixed
    {
        try {
            $result = $fn();
            $this->failureCount = 0;
            return $result;
        } catch (\RedisException $e) {
            $this->failureCount++;
            $this->lastFailureTime = time();

            // 连续失败3次，跳过 Redis 直接降级
            if ($this->failureCount >= 3) {
                return null; // 降级：返回 null 或默认值
            }
            throw $e;
        }
    }
}
```

### 8.4 缓存一致性

```
Cache Aside 模式（最常用）：

读：先读缓存 → 命中返回 → 未命中读 DB → 写缓存 → 返回
写：先写 DB → 删除缓存（不是更新）

    为什么删缓存而不是更新？
    - 更新缓存但没有后续读操作 → 浪费
    - 并发写可能导致缓存写覆盖 → 脏数据

延时双删（强一致性需求）：
1. 先删缓存
2. 更新 DB
3. 延迟 500ms-1s 再删一次缓存
```

```php
final class ProductService
{
    public function updateProduct(int $id, array $data): void
    {
        $key = "product:{$id}";

        // 先删缓存
        $this->redis->del($key);

        // 更新 DB
        DB::table('products')->where('id', $id)->update($data);

        // 延迟再删（可以用消息队列异步做）
        dispatch(fn() => $this->redis->del($key))
            ->delay(now()->addMilliseconds(800));
    }

    public function getProduct(int $id): ?array
    {
        $key = "product:{$id}";

        $cached = $this->redis->get($key);
        if ($cached !== false) return $cached;

        $product = DB::table('products')->find($id);
        if ($product) {
            $this->redis->setex($key, 3600, $product);
        }

        return $product;
    }
}
```

---

## 9. 分布式锁

### 9.1 单节点锁

```php
final class RedisLock
{
    public function __construct(private Redis $redis) {}

    /**
     * 获取锁
     * @return string|null 锁的值（解锁时需要），失败返回 null
     */
    public function acquire(string $key, int $ttl = 10): ?string
    {
        $token = bin2hex(random_bytes(16));

        // SET NX EX 原子操作
        if ($this->redis->set($key, $token, ['nx', 'ex' => $ttl])) {
            return $token;
        }

        return null;
    }

    /**
     * 释放锁（必须判断是自己的锁才能释放）
     */
    public function release(string $key, string $token): bool
    {
        // Lua 保证原子性：先判断，再删除
        $script = <<<'LUA'
if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
else
    return 0
end
LUA;

        return (bool) $this->redis->eval($script, [$key, $token], 1);
    }

    /**
     * 自旋获取锁
     */
    public function acquireWithRetry(string $key, int $ttl = 10, int $maxRetries = 3): ?string
    {
        for ($i = 0; $i < $maxRetries; $i++) {
            $token = $this->acquire($key, $ttl);
            if ($token !== null) return $token;
            usleep(100000); // 100ms
        }
        return null;
    }
}
```

### 9.2 使用示例

```php
$lock = new RedisLock($redis);
$token = $lock->acquire('lock:order:123', 10);

if ($token === null) {
    throw new LockException('操作太频繁，请稍后再试');
}

try {
    // 临界区代码
    $order = $this->placeOrder($data);
} finally {
    $lock->release('lock:order:123', $token);
}
```

### 9.3 RedLock（多节点锁）

```php
// 多个独立的 Redis 实例，半数以上加锁成功才算获取锁
// 适合对一致性要求极高的场景
// 一般业务用单节点 + 主从复制就够了
```

---

## 10. 延迟队列

利用 Sorted Set 的 score 存过期时间戳来实现。

```php
final class DelayQueue
{
    public function __construct(private Redis $redis) {}

    /**
     * 添加延迟任务
     */
    public function push(string $queue, array $data, int $delaySeconds): void
    {
        $executeAt = time() + $delaySeconds;
        $this->redis->zAdd("delay:{$queue}", $executeAt, json_encode($data));
    }

    /**
     * 消费到期任务（定时任务轮询调用）
     */
    public function consume(string $queue, callable $handler): int
    {
        $key = "delay:{$queue}";
        $now = time();
        $count = 0;

        // 取出所有到期的任务
        $tasks = $this->redis->zRangeByScore($key, 0, $now, ['limit' => [0, 100]]);

        if (empty($tasks)) return 0;

        // Lua 保证原子弹出
        $script = <<<'LUA'
local items = redis.call('zrangebyscore', KEYS[1], 0, ARGV[1], 'limit', 0, 100)
if #items > 0 then
    redis.call('zremrangebyscore', KEYS[1], 0, ARGV[1])
end
return items
LUA;

        $items = $this->redis->eval($script, [$key, $now], 1);

        foreach ($items as $item) {
            $handler(json_decode($item, true));
            $count++;
        }

        return $count;
    }
}

// 使用：30分钟后取消未支付订单
$delayQueue = new DelayQueue($redis);
$delayQueue->push('cancel_order', ['order_id' => 123], 1800);

// 定时任务（每分钟执行）
// * * * * * php artisan queue:delay:process
```

---

## 11. 排行榜

### 11.1 实时排行榜

```php
final class Leaderboard
{
    public function __construct(private Redis $redis) {}

    // 更新分数
    public function updateScore(string $board, string $player, int $score): void
    {
        $this->redis->zAdd("board:{$board}", $score, $player);
    }

    // 增加分数
    public function addScore(string $board, string $player, int $delta): int
    {
        return (int) $this->redis->zIncrBy("board:{$board}", $delta, $player);
    }

    // Top N
    public function topN(string $board, int $n = 10): array
    {
        return $this->redis->zRevRange("board:{$board}", 0, $n - 1, true);
    }

    // 玩家排名（从1开始）
    public function rank(string $board, string $player): ?int
    {
        $rank = $this->redis->zRevRank("board:{$board}", $player);
        return $rank !== false ? $rank + 1 : null;
    }

    // 玩家分数
    public function score(string $board, string $player): ?int
    {
        $score = $this->redis->zScore("board:{$board}", $player);
        return $score !== false ? (int) $score : null;
    }

    // 附近排名（玩家前后各5名）
    public function around(string $board, string $player, int $around = 5): array
    {
        $rank = $this->redis->zRevRank("board:{$board}", $player);
        if ($rank === false) return [];

        $start = max(0, $rank - $around);
        $end = $rank + $around;

        return $this->redis->zRevRange("board:{$board}", $start, $end, true);
    }
}
```

### 11.2 多维度排行

```php
// 同分按时间先后排
// score = 分数 * 10^10 + (正序用 当前时间戳 - 操作时间)
// 或 score = 分数 * 10^10 - timestamp

$compositeScore = $score * 1e10 + (time() - $actionTime);
$redis->zAdd('board:daily', $compositeScore, $player);

// 查询时解出原始分数
$raw = $redis->zRevRange('board:daily', 0, 9, true);
foreach ($raw as $player => $composite) {
    $score = (int) ($composite / 1e10);
    echo "{$player}: {$score}\n";
}

// 日/周/月榜：key 名带周期
$redis->zAdd('board:rank:' . date('Ymd'), $score, $player);     // 日榜
$redis->zAdd('board:rank:w' . date('W'), $score, $player);      // 周榜
$redis->zAdd('board:rank:' . date('Ym'), $score, $player);       // 月榜
```

---

## 12. 限流器

### 12.1 固定窗口

```php
/**
 * 固定窗口限流（简单但有临界问题）
 *
 * 缺点：窗口边界的瞬间可能通过 2 × limit 个请求
 * 例如 limit=10，在第59秒和第61秒各发10个 → 2秒内过了20个
 */
final class FixedWindowLimiter
{
    public function __construct(private Redis $redis) {}

    public function isAllowed(string $key, int $limit, int $windowSeconds): bool
    {
        $now = time();
        $window = (int) ($now / $windowSeconds);
        $windowKey = "rate:{$key}:{$window}";

        $count = $this->redis->incr($windowKey);
        $this->redis->expire($windowKey, $windowSeconds * 2); // 冗余过期

        return $count <= $limit;
    }
}
```

### 12.2 滑动窗口（更精确）

```php
/**
 * 滑动窗口限流 — 使用 Sorted Set
 *
 * 精确控制任意1秒/1分钟窗口内的请求数
 */
final class SlidingWindowLimiter
{
    public function __construct(private Redis $redis) {}

    public function isAllowed(string $key, int $limit, int $windowSeconds): bool
    {
        $now = microtime(true);
        $windowStart = $now - $windowSeconds;
        $zsetKey = "rate:sw:{$key}";

        // Lua 实现原子操作
        $script = <<<'LUA'
local key = KEYS[1]
local now = tonumber(ARGV[1])
local windowStart = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local member = ARGV[4]

-- 删除窗口外的旧记录
redis.call('zremrangebyscore', key, 0, windowStart)

-- 统计窗口内请求数
local count = redis.call('zcard', key)
if count < limit then
    redis.call('zadd', key, now, member)
    redis.call('expire', key, tonumber(ARGV[5]))
    return 1
end
return 0
LUA;

        return (bool) $this->redis->eval(
            $script,
            [$zsetKey, $now, $windowStart, $limit, uniqid('', true), $windowSeconds * 2],
            1
        );
    }
}
```

### 12.3 令牌桶（见 5.2 Lua 脚本）

---

## 13. 布隆过滤器

判断一个元素是否**可能存在**（不会漏判，可能误判）。

```php
/**
 * 使用 RedisBloom 模块或手写实现
 *
 * 场景：
 * - 缓存穿透防护：判断 key 是否可能存在
 * - 防止用户推荐重复
 * - URL 去重
 */

// 安装 RedisBloom 模块后
// BF.RESERVE filter:users 0.01 1000000  # 误判率1%, 100万条
// BF.ADD filter:users "user:12345"
// BF.EXISTS filter:users "user:12345"  # 1存在  0不存在

// PHP 手动实现（简化版，多哈希函数）
final class BloomFilter
{
    private int $size;
    private string $key;

    public function __construct(
        private Redis $redis,
        string $name,
        int $expectedElements = 100000,
        float $falsePositiveRate = 0.01,
    ) {
        // 计算位图大小和哈希次数
        $this->size = (int) ceil(
            -$expectedElements * log($falsePositiveRate) / pow(log(2), 2)
        );
        $this->key = "bf:{$name}";
    }

    public function add(string $element): void
    {
        foreach ($this->hashOffsets($element, 7) as $offset) {
            $this->redis->setBit($this->key, $offset, 1);
        }
    }

    public function exists(string $element): bool
    {
        foreach ($this->hashOffsets($element, 7) as $offset) {
            if (!$this->redis->getBit($this->key, $offset)) {
                return false; // 一定不存在
            }
        }
        return true; // 可能存在
    }

    private function hashOffsets(string $element, int $hashCount): array
    {
        $offsets = [];
        for ($i = 0; $i < $hashCount; $i++) {
            $hash = crc32($element . $i);
            $offsets[] = abs($hash) % $this->size;
        }
        return $offsets;
    }
}

// 使用：缓存穿透防护
$bf = new BloomFilter($redis, 'users', 1000000);
$bf->add('user:12345');

// 查询时先判断
if (!$bf->exists("user:{$id}")) {
    return null; // 一定不存在，直接返回
}
// 可能存在，查缓存 → 数据库
```

---

## 14. 会话与 Token 管理

```php
final class TokenManager
{
    public function __construct(private Redis $redis) {}

    /**
     * 生成 Token（支持多端登录）
     */
    public function createToken(int $userId, string $device = 'web'): string
    {
        $token = bin2hex(random_bytes(32));
        $key = "token:{$token}";
        $userKey = "user:{$userId}:tokens";

        // 存储 token → user 映射
        $this->redis->hMSet($key, [
            'user_id' => $userId,
            'device'  => $device,
            'login_at' => time(),
            'ip'      => request()->ip(),
        ]);
        $this->redis->expire($key, 86400 * 7); // 7天过期

        // 记录用户的所有 token（用于踢下线）
        $this->redis->sAdd($userKey, $token);
        $this->redis->expire($userKey, 86400 * 30);

        return $token;
    }

    /**
     * 验证 Token
     */
    public function validate(string $token): ?array
    {
        $key = "token:{$token}";
        $data = $this->redis->hGetAll($key);

        if (empty($data)) return null;

        // 滑动过期：每次验证成功续期
        $this->redis->expire($key, 86400 * 7);

        return $data;
    }

    /**
     * 踢掉某用户所有设备
     */
    public function revokeAll(int $userId): void
    {
        $userKey = "user:{$userId}:tokens";
        $tokens = $this->redis->sMembers($userKey);

        foreach ($tokens as $token) {
            $this->redis->del("token:{$token}");
        }

        $this->redis->del($userKey);
    }

    /**
     * 刷新 Token（旧 Token 换新）
     */
    public function refresh(string $oldToken): ?string
    {
        $data = $this->validate($oldToken);
        if (!$data) return null;

        // 删除旧 Token
        $this->redis->del("token:{$oldToken}");
        $this->redis->sRem("user:{$data['user_id']}:tokens", $oldToken);

        // 创建新 Token
        return $this->createToken($data['user_id'], $data['device']);
    }
}
```

---

## 15. 计数器与统计

```php
final class CounterService
{
    public function __construct(private Redis $redis) {}

    /**
     * 文章浏览量（Hash 存储，减少 key 数量）
     */
    public function recordView(int $articleId): void
    {
        $this->redis->hIncrBy('stats:articles', (string) $articleId, 1);
    }

    public function getView(int $articleId): int
    {
        return (int) $this->redis->hGet('stats:articles', (string) $articleId);
    }

    /**
     * 批量同步到 DB
     */
    public function syncToDB(): void
    {
        $data = $this->redis->hGetAll('stats:articles');
        foreach ($data as $articleId => $views) {
            DB::table('articles')
                ->where('id', $articleId)
                ->increment('views', $views);
        }
        $this->redis->del('stats:articles');
    }

    /**
     * 每小时 PV 统计 (Bitmap + 时间维度的 Hash)
     */
    public function recordPV(int $pageId): void
    {
        $hour = date('YmdH');
        $this->redis->hIncrBy("pv:{$pageId}", $hour, 1);
        $this->redis->expire("pv:{$pageId}", 86400 * 30);
    }

    /**
     * 今日 UV (HyperLogLog)
     */
    public function recordUV(int $pageId, string $userId): void
    {
        $date = date('Ymd');
        $this->redis->pfAdd("uv:{$pageId}:{$date}", $userId);
    }

    public function getTodayUV(int $pageId): int
    {
        $date = date('Ymd');
        return $this->redis->pfCount("uv:{$pageId}:{$date}");
    }

    /**
     * 漏斗统计（用户行为转化率）
     */
    public function recordFunnel(string $step, int $userId): void
    {
        $date = date('Ymd');
        $this->redis->pfAdd("funnel:{$date}:{$step}", $userId);
    }

    public function getFunnelRates(string $date): array
    {
        $steps = ['visit', 'add_cart', 'create_order', 'pay'];
        $rates = [];

        foreach ($steps as $step) {
            $count = $this->redis->pfCount("funnel:{$date}:{$step}");
            $rates[$step] = $count;
        }

        return $rates;
        // ['visit' => 10000, 'add_cart' => 2000, 'create_order' => 500, 'pay' => 300]
        // 转化率: 20% → 25% → 60%
    }
}
```

---

## 16. 消息队列替代方案

Redis 做轻量级消息队列（不需要 RabbitMQ/Kafka 时）：

```php
final class RedisQueue
{
    /**
     * 生产者
     */
    public function push(string $queue, array $data): void
    {
        $this->redis->lPush(
            "queue:{$queue}",
            json_encode($data)
        );
    }

    /**
     * 可靠消费者（阻塞 + 备份队列防丢失）
     */
    public function consume(string $queue, callable $handler): void
    {
        $backupKey = "queue:{$queue}:backup";

        while (true) {
            // BRPOPLPUSH：原子弹出 + 备份
            $item = $this->redis->brPopLPush(
                "queue:{$queue}", $backupKey, 5
            );

            if ($item === null) continue; // 超时无数据

            try {
                $handler(json_decode($item, true));
                // 处理成功，从备份队列删除
                $this->redis->lRem($backupKey, $item, 1);
            } catch (\Throwable $e) {
                Log::error('队列处理失败', [
                    'queue' => $queue,
                    'item' => $item,
                    'error' => $e->getMessage(),
                ]);
                // 失败的任务留在 backup 队列，人工/定时重试
            }
        }
    }

    /**
     * 重试失败任务
     */
    public function retryFailed(string $queue): void
    {
        $backupKey = "queue:{$queue}:backup";

        while ($item = $this->redis->rPop($backupKey)) {
            $this->redis->lPush("queue:{$queue}", $item);
        }
    }
}
```

Stream 类型（Redis 5.0+，更正式的消息队列）：

```php
// 生产
$redis->xAdd('stream:orders', '*', [
    'order_id' => 123,
    'action'   => 'paid',
    'amount'   => 99.00,
]);

// 消费组
$redis->xGroup('create', 'stream:orders', 'order-processors', 0, true);

// 消费
$messages = $redis->xReadGroup('order-processors', 'worker-1', ['stream:orders' => '>'], 1, 5000);
foreach ($messages['stream:orders'] as $id => $fields) {
    processOrder($fields);
    $redis->xAck('stream:orders', 'order-processors', $id);
}
```

---

## 17. 运维与监控

### 17.1 慢查询

```bash
# redis.conf
slowlog-log-slower-than 10000  # 超过10毫秒记录
slowlog-max-len 128

# 查看
redis> SLOWLOG GET 10
```

### 17.2 内存优化

```php
// hash-max-ziplist-entries 512 — 小 Hash 用压缩列表
// hSet 比多个 set 更省内存，尤其是存储大量小对象的场景

// ❌ 内存浪费
$redis->set('user:1:name', 'khz');
$redis->set('user:1:email', 'khz@example.com');
$redis->set('user:1:phone', '13800138000');
// 每个 key 都有元数据开销

// ✅ 内存友好（Hash 压缩存储）
$redis->hMSet('user:1', [
    'name'  => 'khz',
    'email' => 'khz@example.com',
    'phone' => '13800138000',
]);
```

### 17.3 键命名约定

```
业务:模块:标识:字段

user:100:info             Hash  用户信息
user:100:follows          Set   关注列表
user:100:followers        Set   粉丝列表
cache:product:100         String 商品缓存
lock:order:123            String 订单锁
rate:login:13800138000    String 登录频率
queue:email               List  邮件队列
board:daily:20260513      ZSet  每日排行
counter:article:views     String 文章浏览
session:xxx               String Session
token:xxx                 Hash   Token信息
```

### 17.4 常用运维命令

```bash
# 连接 & 信息
redis-cli -h 127.0.0.1 -p 6379 -a password
INFO                     # 服务器全部信息
INFO memory              # 内存使用
INFO stats               # 统计信息（命中率等）
INFO clients             # 客户端连接
DBSIZE                   # 当前库 key 数量
CONFIG GET maxmemory     # 查看配置

# Key 管理
KEYS user:*              # ⚠️ 生产禁用！用 SCAN
SCAN 0 MATCH user:* COUNT 100  # 安全遍历
TYPE key                 # key 类型
OBJECT ENCODING key      # 底层编码
MEMORY USAGE key         # key 占用内存

# 过期
TTL key                  # 剩余秒数
PTTL key                 # 剩余毫秒（-1永不过期, -2不存在）
EXPIRE key 300           # 设置5分钟过期
PERSIST key              # 移除过期

# 监控
MONITOR                  # 实时监控所有命令 ⚠️ 影响性能
CLIENT LIST              # 客户端列表
```

---

> 📝 **Redis 是最趁手的瑞士军刀，但别把它当成唯一的数据存储。核心数据落 DB，Redis 做辅助和加速。**
