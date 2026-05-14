# RabbitMQ 消息队列实战（PHP 版）

> 从交换机到死信队列，完整的异步消息解决方案

---

## 目录

1. [消息队列基础](#1-消息队列基础)
2. [RabbitMQ 核心概念](#2-rabbitmq-核心概念)
3. [安装与连接](#3-安装与连接)
4. [六种工作模式](#4-六种工作模式)
5. [消息确认机制](#5-消息确认机制)
6. [持久化与可靠性](#6-持久化与可靠性)
7. [死信队列](#7-死信队列)
8. [延迟队列](#8-延迟队列)
9. [TTL 与优先级](#9-ttl-与优先级)
10. [PHP 消费端最佳实践](#10-php-消费端最佳实践)
11. [生产端最佳实践](#11-生产端最佳实践)
12. [Spring AMQP vs php-amqplib 对比](#12-spring-amqp-vs-php-amqplib-对比)
13. [实战场景](#13-实战场景)
14. [监控与运维](#14-监控与运维)
15. [RocketMQ 实战](#15-rocketmq-实战)
16. [Kafka 实战](#16-kafka-实战)

---

## 1. 消息队列基础

### 1.1 为什么需要消息队列

```
核心场景：

1. 削峰填谷
   秒杀峰 10000 请求/秒 → 队列缓冲 → MySQL/订单处理 1000/秒
   ┌──────┐     ┌──────┐     ┌──────┐
   │ 请求  │────→│ 队列  │────→│ 处理  │
   │ 10000│     │ 缓冲  │     │ 1000 │
   └──────┘     └──────┘     └──────┘

2. 异步解耦
   下订单后不需要同步等短信/邮件/库存更新
   订单系统 → 队列 → 短信服务 / 邮件服务 / 库存服务

3. 日志收集
   N 个 App 写日志到队列 → 统一消费 → 写入 ES/数据库
```

### 1.2 主流消息中间件横向对比

| 特性 | RabbitMQ | RocketMQ | Kafka | Redis Stream |
|------|----------|----------|-------|-------------|
| **语言** | Erlang | Java | Java/Scala | C |
| **协议** | AMQP 0-9-1 | 自定义 | 自定义 | RESP |
| **吞吐量** | 万级/秒 | 十万级/秒 | 百万级/秒 | 十万级/秒 |
| **延迟** | 微秒级(μs) | 毫秒级 | 毫秒级 | 微秒级 |
| **持久化** | ✅ 磁盘持久 | ✅ 同步刷盘 | ✅ PageCache | ✅ AOF/RDB |
| **消费确认** | ✅ ACK | ✅ ACK+重试 | ✅ Offset Commit | ✅ ACK/XACK |
| **消息回溯** | ❌ | ✅ 按时间/偏移 | ✅ 按时间/偏移 | ❌ (消费即删) |
| **延时消息** | ✅ 插件 | ✅ 18个级别 | ❌ | ❌ |
| **事务消息** | ❌ | ✅ 支持 | ✅ 幂等 | ❌ |
| **顺序消息** | 单队列顺序 | ✅ 全局/分区有序 | ✅ 分区有序 | ❌ |
| **批量发送** | ✅ batch_publish | ✅ 原生支持 | ✅ 原生支持 | ✅ PIPELINE |
| **路由能力** | 极灵活(Exchange+Binding) | Tag 过滤 | 分区路由 | Consumer Group |
| **消费模型** | Push 为主 | Pull/Push | Pull 为主 | Pull |
| **消息过滤** | ❌ | ✅ SQL92/Tag | ❌ | ❌ |
| **重试机制** | Nack+死信 | 原生重试18次 | Offset 控制 | XACK+重入队 |
| **运维复杂度** | 中 | 高 | 高 | 低(复用Redis) |
| **PHP 生态** | php-amqplib | rocketmq-client-php | rdkafka+librdkafka | Redis扩展 |
| **核心场景** | 企业消息总线 | 电商交易/金融 | 大数据/日志流 | 轻量异步 |

### 1.3 选型指南

```
选型决策树：

需要绝对可靠 + 复杂路由？
  → RabbitMQ（AMQP 标准、Exchange 灵活路由、死信/延迟）

电商交易/金融场景 + 事务消息 + 顺序消息？
  → RocketMQ（阿里双11验证、18级延迟、分布式事务）

海量日志/埋点/大数据流 + 高吞吐？
  → Kafka（百万QPS、顺序IO、消息回溯、流计算生态）

已有 Redis + 轻量级异步？
  → Redis Stream（零新增组件、Stream 有 ACK 比 Pub/Sub 可靠）

PHP 项目推荐：
  - 一般企业应用：RabbitMQ（生态成熟、php-amqplib 稳定）
  - 高并发交易：RocketMQ（事务消息强需求时）
  - 日志/监控数据管道：Kafka
  - 简单的异步任务：Redis Stream / Laravel Queue + Redis
```
| **推送/拉取** | 推为主 | 拉为主 | 推 |
| **运维复杂度** | 中 | 高 | 低 |

---

## 2. RabbitMQ 核心概念

```
RabbitMQ 是基于 AMQP 协议的消息中间件

架构：
┌──────────────────────────────────────────────┐
│                 RabbitMQ Broker                │
│                                                │
│  ┌──────────────────────────────────────┐     │
│  │              Virtual Host              │     │
│  │  ┌──────────┐       ┌──────────┐     │     │
│  │  │ Exchange  │──Bind──│  Queue    │     │     │
│  │  │  (交换机)  │       │  (队列)    │     │     │
│  │  └────┬─────┘       └────┬─────┘     │     │
│  │       │                  │            │     │
│  └───────┼──────────────────┼────────────┘     │
└──────────┼──────────────────┼──────────────────┘
           │                  │
   Producer (生产)       Consumer (消费)
```

```
核心组成：

Producer（生产者）
  → 发送消息的应用程序

Exchange（交换机）
  → 接收消息，按路由规则分发到队列
  → 类型：direct / topic / fanout / headers

Binding（绑定）
  → 交换机和队列之间的路由规则
  → Routing Key + Exchange → 匹配 → 放入哪个队列

Queue（队列）
  → 消息的存储容器
  → 特性：持久化 / 排他 / 自动删除

Consumer（消费者）
  → 从队列获取消息的应用程序
  → 支持 Push（推送）和 Pull（拉取）

Connection（连接）
  → TCP 长连接，复用一个连接创建多个 Channel

Channel（信道）
  → 轻量级连接，在同一个 TCP 连接内的逻辑连接
  → 复用 TCP 连接，减少握手开销

Virtual Host（虚拟主机）
  → 逻辑隔离，类似 MySQL 的 database
```

---

## 3. 安装与连接

```bash
# Docker 安装
docker run -d --name rabbitmq \
  -p 5672:5672 \        # AMQP 协议
  -p 15672:15672 \      # Web 管理界面
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=admin123 \
  rabbitmq:3-management

# PHP 依赖
composer require php-amqplib/php-amqplib
```

```php
// ═══ 连接工厂 ═══
final class RabbitMQFactory
{
    private static ?PhpAmqpLib\Connection\AMQPStreamConnection $connection = null;

    public static function getChannel(): PhpAmqpLib\Channel\AMQPChannel
    {
        if (self::$connection === null || !self::$connection->isConnected()) {
            self::$connection = new PhpAmqpLib\Connection\AMQPStreamConnection(
                'localhost',    // host
                5672,           // port
                'admin',        // user
                'admin123',     // password
                '/',            // vhost
                false,          // insist
                'AMQPLAIN',     // login_method
                null,           // login_response
                'en_US',        // locale
                3.0,            // connection_timeout
                3.0,            // read_write_timeout
                null,           // context
                false,          // keepalive
                60              // heartbeat
            );
        }

        return self::$connection->channel();
    }
}
```

---

## 4. 六种工作模式

### 4.1 Simple（简单模式）

```
P ────→ [Queue] ────→ C
```

```php
// ═══ 发送 ═══
$channel = RabbitMQFactory::getChannel();
$channel->queue_declare('hello', false, false, false, false);
//                        队列名  持久化 排他 自动删除 参数

$msg = new PhpAmqpLib\Message\AMQPMessage('Hello World!');
$channel->basic_publish($msg, '', 'hello');
//                        消息  交换机  路由key

$channel->close();

// ═══ 消费 ═══
$channel = RabbitMQFactory::getChannel();
$channel->queue_declare('hello', false, false, false, false);

echo "等待消息...\n";
$callback = function ($msg) {
    echo "收到: {$msg->body}\n";
};

$channel->basic_consume('hello', '', false, true, false, false, $callback);

while ($channel->is_consuming()) {
    $channel->wait();
}
```

### 4.2 Work Queue（竞争消费）

```
P ───→ [Queue] ──┬──→ C1
                  └──→ C2
```

```php
// ═══ 发送（多消息） ═══
$channel = RabbitMQFactory::getChannel();
$channel->queue_declare('tasks', false, true, false, false);
//                            持久化  不排他 不自动删除

for ($i = 1; $i <= 10; $i++) {
    $msg = new PhpAmqpLib\Message\AMQPMessage("Task #{$i}", [
        'delivery_mode' => PhpAmqpLib\Message\AMQPMessage::DELIVERY_MODE_PERSISTENT,
    ]);
    $channel->basic_publish($msg, '', 'tasks');
}

// ═══ 公平调度（公平分发，处理完再给下一条） ═══
$channel->basic_qos(null, 1, null);  // 每次只取 1 条
//              prefetch_size=null prefetch_count=1 global=null

$callback = function ($msg) {
    echo "处理: {$msg->body}\n";
    sleep(2);  // 模拟耗时
    $msg->ack();  // ✅ 手动确认
};

$channel->basic_consume('tasks', '', false, false, false, false, $callback);
//                                     no_ack=false 手动确认
```

### 4.3 Publish/Subscribe（发布订阅）

```
P ───→ [Exchange (fanout)] ──┬──→ Queue1 ──→ C1
                              ├──→ Queue2 ──→ C2
                              └──→ Queue3 ──→ C3
```

```php
// ═══ 发送 ═══
$channel = RabbitMQFactory::getChannel();
$channel->exchange_declare('logs', 'fanout', false, true, false);
//                         交换机名  类型    持久化  自动删除

$msg = new PhpAmqpLib\Message\AMQPMessage('广播消息');
$channel->basic_publish($msg, 'logs', '');  // routing_key 在 fanout 里被忽略

// ═══ 消费（每个消费者有自己的临时队列） ═══
$channel = RabbitMQFactory::getChannel();
$channel->exchange_declare('logs', 'fanout', false, true, false);

// 随机队列名（消费者断开后自动删除）
[$queueName] = $channel->queue_declare('', false, false, true, true);
//                                      不持久化 不排他 自动删除

$channel->queue_bind($queueName, 'logs', '');

$channel->basic_consume($queueName, '', false, true, false, false, function ($msg) {
    echo "收到: {$msg->body}\n";
});
```

### 4.4 Routing（路由模式）

```
P ───→ [Exchange (direct)] ──┬──→ [error]   ──→ C1(只消费error)
                              ├──→ [info]    ──→ C2
                              └──→ [warning] ──→ C3
```

```php
// ═══ 发送（不同路由级别） ═══
$channel = RabbitMQFactory::getChannel();
$channel->exchange_declare('logs_direct', 'direct', false, true, false);

$levels = ['error', 'info', 'warning'];
for ($i = 0; $i < 10; $i++) {
    $level = $levels[array_rand($levels)];
    $msg = new PhpAmqpLib\Message\AMQPMessage("[{$level}] 日志消息 {$i}");
    $channel->basic_publish($msg, 'logs_direct', $level);
}

// ═══ 消费（只消费 error 和 warning） ═══
$channel->exchange_declare('logs_direct', 'direct', false, true, false);
[$queueName] = $channel->queue_declare('', false, false, true, true);

// 绑定多个 routing key
$channel->queue_bind($queueName, 'logs_direct', 'error');
$channel->queue_bind($queueName, 'logs_direct', 'warning');
```

### 4.5 Topics（主题模式）

```
P ─→ [Exchange (topic)] ─┬→ *.orange.* → Queue1
                          ├→ *.*.rabbit → Queue2
                          └→ lazy.#     → Queue3

例：quick.orange.rabbit → Queue1 + Queue2
    lazy.orange.elephant → Queue1 + Queue3
    quick.orange.fox → Queue1
    lazy.brown.fox → Queue3
    quick.brown.fox → 丢弃
```

```php
// ═══ 发送 ═══
$channel = RabbitMQFactory::getChannel();
$channel->exchange_declare('logs_topic', 'topic', false, true, false);

$routingKeys = [
    'quick.orange.rabbit',
    'lazy.orange.elephant',
    'quick.orange.fox',
    'lazy.brown.fox',
    'quick.brown.fox',
];
foreach ($routingKeys as $key) {
    $msg = new PhpAmqpLib\Message\AMQPMessage("消息: {$key}");
    $channel->basic_publish($msg, 'logs_topic', $key);
}

// ═══ 消费：所有以 lazy 开头的消息路由 ═══
$channel->exchange_declare('logs_topic', 'topic', false, true, false);
[$queueName] = $channel->queue_declare('', false, false, true, true);
$channel->queue_bind($queueName, 'logs_topic', 'lazy.#');
```

### 4.6 RPC（远程调用）

```
Client ───→ [request_queue] ───→ Server（处理）
Client ←── (reply_to) ←──────── Server（返回结果）
```

```php
// ═══ Server ═══
$channel = RabbitMQFactory::getChannel();
$channel->queue_declare('rpc_queue', false, false, false, false);
$channel->basic_qos(null, 1, null);

$channel->basic_consume('rpc_queue', '', false, false, false, false,
    function ($msg) use ($channel) {
        $body = $msg->body;
        echo "处理: {$body}\n";

        // 计算结果
        $result = strlen($body);  // 模拟计算

        // 回复到 reply_to 队列
        $reply = new PhpAmqpLib\Message\AMQPMessage(
            (string)$result,
            ['correlation_id' => $msg->get('correlation_id')]
        );
        $channel->basic_publish($reply, '', $msg->get('reply_to'));
        $msg->ack();
    }
);

// ═══ Client ═══
$channel = RabbitMQFactory::getChannel();

// 随机回调队列
[$callbackQueue] = $channel->queue_declare('', false, false, true, false);

$corrId = uniqid();
$response = null;

$channel->basic_consume($callbackQueue, '', false, true, false, false,
    function ($msg) use ($corrId, &$response) {
        if ($msg->get('correlation_id') === $corrId) {
            $response = $msg->body;
        }
    }
);

$msg = new PhpAmqpLib\Message\AMQPMessage(
    'Hello RPC',
    ['correlation_id' => $corrId, 'reply_to' => $callbackQueue]
);
$channel->basic_publish($msg, '', 'rpc_queue');

// 等待回复
while ($response === null) {
    $channel->wait(null, false, 10);
}

echo "RPC 结果: {$response}\n";
```

### 4.7 模式对照表

| 模式 | Exchange | Routing Key | 适用场景 |
|------|----------|-------------|---------|
| Simple | '' (默认) | queue名 | 单发单收 |
| Work Queue | '' (默认) | queue名 | 任务分发，多消费者竞争 |
| Pub/Sub | fanout | (忽略) | 广播通知 |
| Routing | direct | 精确 key | 按等级分发日志 |
| Topics | topic | 通配 pattern | 灵活路由，多维度匹配 |
| RPC | (任意) | (任意) | 远程调用，请求-响应 |

---

## 5. 消息确认机制

### 5.1 Publisher Confirm（生产端确认）

```php
$channel = RabbitMQFactory::getChannel();
$channel->confirm_select();  // 开启发布确认模式

$msg = new PhpAmqpLib\Message\AMQPMessage('重要消息');
$channel->basic_publish($msg, 'my_exchange', 'my_key');

// 同步等待确认
$channel->wait_for_pending_acks(5);  // 超时 5 秒

// 或异步
$channel->set_ack_handler(function () {
    echo "消息确认发送成功\n";
});
$channel->set_nack_handler(function () {
    echo "消息发送失败\n";
});
$channel->wait_for_pending_acks();
```

### 5.2 Consumer ACK（消费确认）

```php
// ═══ 手动 ACK 三种方式 ═══

$callback = function ($msg) {
    try {
        // 处理消息
        processOrder($msg->body);

        // ✅ 单条确认
        $msg->ack();

        // ✅ 拒绝单条（重新入队）
        // $msg->nack(true);   (requeue=true)

        // ✅ 拒绝单条（丢弃）
        // $msg->nack(false);  (requeue=false)

    } catch (\Throwable $e) {
        // 处理异常：拒绝并丢弃（或进入死信）
        error_log("处理失败: {$e->getMessage()}");
        $msg->nack(false);  // 不重新入队，进死信或丢弃
    }
};

$channel->basic_qos(null, 1, null);
$channel->basic_consume('orders', '', false, false, false, false, $callback);
//                                 no_ack=false → 手动确认

// ═══ 拒绝多条消息 ═══
// $channel->basic_nack($deliveryTag, $multiple=true, $requeue=false);
```

---

## 6. 持久化与可靠性

```php
// ═══ 三重保障（缺一不可） ═══

// 1. 队列持久化
$channel->queue_declare('orders', false, true, false, false);
//                               passive=false, durable=true

// 2. 消息持久化
$msg = new PhpAmqpLib\Message\AMQPMessage($data, [
    'delivery_mode' => PhpAmqpLib\Message\AMQPMessage::DELIVERY_MODE_PERSISTENT,
]);

// 3. 交换机持久化
$channel->exchange_declare('orders_x', 'direct', false, true, false);
//                                              passive, durable=true

// ⚠️ 持久化 != 实时落盘（fsync 有间隔）
// ⚠️ 集群 + 镜像队列才能高可用
```

---

## 7. 死信队列（DLX）

```
死信队列 = 处理失败消息的兜底机制

消息变成死信的条件：
1. 被拒绝（reject/nack）且 requeue=false
2. 队列消息 TTL 过期
3. 队列达到最大长度

流程：
  [原始队列] ──(死信)──→ [死信交换机] ──→ [死信队列] ──→ [告警/重试]
```

```php
// ═══ 声明死信交换机 ═══
$channel = RabbitMQFactory::getChannel();
$channel->exchange_declare('dlx_exchange', 'direct', false, true, false);
$channel->queue_declare('dead_letter_queue', false, true, false, false);
$channel->queue_bind('dead_letter_queue', 'dlx_exchange', 'dead');

// ═══ 原始队列绑定死信 ═══
$channel->queue_declare('orders', false, true, false, false, false, [
    'x-dead-letter-exchange'    => ['S', 'dlx_exchange'],
    'x-dead-letter-routing-key' => ['S', 'dead'],
    'x-message-ttl'             => ['I', 30000],  // 30秒未消费变死信
    'x-max-length'              => ['I', 10000],   // 最多存1万条
]);

// ═══ 消费死信队列（做告警/记录/人工处理） ═══
$channel->basic_consume('dead_letter_queue', '', false, false, false, false,
    function ($msg) {
        $headers = $msg->get('application_headers')->getNativeData();
        $reason = $headers['x-death'][0]['reason'] ?? '未知';

        // 记录到数据库
        error_log("死信消息 [{$reason}]: {$msg->body}");

        // 发告警通知
        sendAlert("死信消息: {$msg->body}");

        $msg->ack();
    }
);
```

---

## 8. 延迟队列

```
延迟队列 = 消息发送后，过指定时间才投递给消费者

两种实现：

方案1：死信 + TTL（推荐，无需插件）
  [延迟交换机] → [延迟队列 (TTL=10s)] → (过期变死信) → [目标队列]
                                                  ↑ DLX 配置指向目标

方案2：rabbitmq-delayed-message-exchange 插件
  docker run -d rabbitmq:3-management
  docker exec rabbitmq rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

```php
// ═══ 方案1：TTL + DLX 实现延迟队列 ═══
final class DelayQueue
{
    private PhpAmqpLib\Channel\AMQPChannel $channel;

    public function __construct()
    {
        $this->channel = RabbitMQFactory::getChannel();
    }

    // 声明延迟队列（不同延迟用不同队列）
    public function declareDelayQueue(string $name, int $delayMs): void
    {
        // 死信交换机（作为延迟后的目标）
        $this->channel->exchange_declare("{$name}_dlx", 'direct', false, true, false);
        $this->channel->queue_declare("{$name}_result", false, true, false, false);
        $this->channel->queue_bind("{$name}_result", "{$name}_dlx", "{$name}");

        // 延迟队列（消息在这里等到过期）
        $this->channel->queue_declare($name, false, true, false, false, false, [
            'x-dead-letter-exchange'    => ['S', "{$name}_dlx"],
            'x-dead-letter-routing-key' => ['S', $name],
            'x-message-ttl'             => ['I', $delayMs],
        ]);
    }

    // 发送延迟消息
    public function publish(string $queue, string $message): void
    {
        $msg = new PhpAmqpLib\Message\AMQPMessage($message, [
            'delivery_mode' => PhpAmqpLib\Message\AMQPMessage::DELIVERY_MODE_PERSISTENT,
        ]);
        $this->channel->basic_publish($msg, '', $queue);
    }

    // 消费延迟后的消息
    public function consume(string $name, callable $callback): void
    {
        $this->channel->basic_qos(null, 1, null);
        $this->channel->basic_consume(
            "{$name}_result", '', false, false, false, false, $callback
        );
    }
}

// ═══ 使用 ═══
$delay = new DelayQueue();

// 声明：30秒后推送通知
$delay->declareDelayQueue('notification_30s', 30000);

// 发送
$delay->publish('notification_30s', json_encode([
    'user_id' => 123,
    'title'   => '订单即将超时',
]));

// 消费
$delay->consume('notification_30s', function ($msg) {
    $data = json_decode($msg->body, true);
    sendPushNotification($data['user_id'], $data['title']);
    $msg->ack();
});
```

---

## 9. TTL 与优先级

```php
// ═══ 消息级 TTL ═══
$msg = new PhpAmqpLib\Message\AMQPMessage('过期消息', [
    'expiration' => '10000',  // 10秒过期
]);

// ═══ 队列级 TTL ═══
$channel->queue_declare('ttl_queue', false, true, false, false, false, [
    'x-message-ttl' => ['I', 60000],  // 队列中所有消息 60 秒过期
]);

// ═══ 优先级队列 ═══
$channel->queue_declare('priority_queue', false, true, false, false, false, [
    'x-max-priority' => ['I', 10],
]);

// 发送高优先级消息
$msg = new PhpAmqpLib\Message\AMQPMessage('VIP消息', [
    'priority' => 10,
]);
$channel->basic_publish($msg, '', 'priority_queue');

// ⚠️ 优先级队列会消耗更多资源
```

---

## 10. PHP 消费端最佳实践

### 10.1 常驻消费脚本（Swoole 协程版）

```php
// bin/consume.php
use Swoole\Coroutine;

final class OrderConsumer
{
    public function run(): void
    {
        Coroutine\run(function () {
            $channel = RabbitMQFactory::getChannel();

            $channel->queue_declare('orders', false, true, false, false);
            $channel->basic_qos(null, 5, null);  // 一次取5条

            $channel->basic_consume('orders', '', false, false, false, false,
                function ($msg) {
                    // 协程处理
                    Coroutine::create(function () use ($msg) {
                        try {
                            $order = json_decode($msg->body, true);
                            $this->processOrder($order);
                            $msg->ack();
                        } catch (\Throwable $e) {
                            error_log("处理失败 #{$msg->getDeliveryTag()}: {$e->getMessage()}");
                            $msg->nack(false);  // 不进死信，直接丢弃
                        }
                    });
                }
            );

            // 信号处理（优雅退出）
            Coroutine::create(function () {
                pcntl_signal(SIGTERM, function () {
                    echo "收到 SIGTERM，准备退出...\n";
                    $GLOBALS['running'] = false;
                });
                pcntl_signal(SIGINT, function () {
                    echo "收到 SIGINT，准备退出...\n";
                    $GLOBALS['running'] = false;
                });
            });

            // 主循环
            while ($channel->is_consuming() && ($GLOBALS['running'] ?? true)) {
                $channel->wait();
            }

            echo "消费结束\n";
            $channel->close();
        });
    }

    private function processOrder(array $order): void
    {
        // 实际业务处理
        $db = new PDO('mysql:host=127.0.0.1;dbname=mydb', 'root', 'pass');
        $stmt = $db->prepare('INSERT INTO orders_log (...) VALUES (...)');
        $stmt->execute([$order['id']]);
    }
}

(new OrderConsumer())->run();
```

### 10.2 断线重连

```php
final class RobustConsumer
{
    private int $reconnectDelay = 1;

    public function start(): void
    {
        while (true) {
            try {
                $this->consume();
            } catch (PhpAmqpLib\Exception\AMQPRuntimeException $e) {
                error_log("连接断开: {$e->getMessage()}");
            } catch (\Throwable $e) {
                error_log("消费异常: {$e->getMessage()}");
            }

            // 指数退避重连
            echo "{$this->reconnectDelay}秒后重连...\n";
            sleep($this->reconnectDelay);
            $this->reconnectDelay = min($this->reconnectDelay * 2, 60);
        }
    }

    private function consume(): void
    {
        $channel = RabbitMQFactory::getChannel();

        // 监听 channel 关闭事件
        $channel->set_close_handler(function () {
            throw new \RuntimeException('Channel 关闭');
        });

        // ... 正常消费逻辑 ...

        while ($channel->is_consuming()) {
            $channel->wait();
        }
    }
}

// 用 Supervisor 守护（生产环境推荐）
// /etc/supervisor/conf.d/order-consumer.conf
// [program:order-consumer]
// command=php /var/www/bin/consume.php
// autostart=true
// autorestart=true
// numprocs=3
```

### 10.3 进程守护（Supervisor）

```ini
; /etc/supervisor/conf.d/consumers.conf

[program:order-consumer]
command=php /var/www/bin/consume_orders.php
directory=/var/www
user=www-data
autostart=true
autorestart=true
numprocs=3                           ; 3个进程并行消费
process_name=%(program_name)s_%(process_num)02d
stdout_logfile=/var/log/order_consumer.log
stderr_logfile=/var/log/order_consumer_error.log
```

---

## 11. 生产端最佳实践

### 11.1 可靠发送

```php
final class ReliablePublisher
{
    public function publish(string $exchange, string $routingKey, array $data): bool
    {
        $channel = RabbitMQFactory::getChannel();
        $channel->confirm_select();

        $msg = new PhpAmqpLib\Message\AMQPMessage(
            json_encode($data),
            [
                'delivery_mode' => PhpAmqpLib\Message\AMQPMessage::DELIVERY_MODE_PERSISTENT,
                'content_type'  => 'application/json',
                'timestamp'     => time(),
                'message_id'    => uniqid('msg_', true),
            ]
        );

        $channel->basic_publish($msg, $exchange, $routingKey);

        try {
            $channel->wait_for_pending_acks(5);
            return true;
        } catch (\Throwable $e) {
            error_log("发布失败: {$e->getMessage()}");
            // 失败保存到数据库，定时重试
            $this->saveToRetryTable($exchange, $routingKey, $data);
            return false;
        }
    }

    private function saveToRetryTable(string $exchange, string $routingKey, array $data): void
    {
        $db = new PDO('mysql:host=127.0.0.1;dbname=mydb', 'root', 'pass');
        $stmt = $db->prepare(
            'INSERT INTO mq_retry (exchange, routing_key, body, created_at) VALUES (?, ?, ?, NOW())'
        );
        $stmt->execute([$exchange, $routingKey, json_encode($data)]);
    }
}
```

### 11.2 批量发送

```php
// ═══ 批量发布 ═══
$channel = RabbitMQFactory::getChannel();
$channel->confirm_select();

$batchSize = 100;
$messages = [
    ['id' => 1, 'user_id' => 100],
    ['id' => 2, 'user_id' => 101],
    // ... 几千条 ...
];

foreach (array_chunk($messages, $batchSize) as $batch) {
    foreach ($batch as $row) {
        $msg = new PhpAmqpLib\Message\AMQPMessage(json_encode($row), [
            'delivery_mode' => PhpAmqpLib\Message\AMQPMessage::DELIVERY_MODE_PERSISTENT,
        ]);
        $channel->batch_basic_publish($msg, 'my_exchange', 'my_key');
    }

    // 一次性发送整批
    $channel->publish_batch();
    $channel->wait_for_pending_acks(10);
}
```

---

## 12. Spring AMQP vs php-amqplib 对比

```
Spring AMQP（Java）          php-amqplib（PHP）

✅ 注解驱动简洁              ⚠️ 手写 Channel/Message 管理
✅ @RabbitListener          ⚠️ basic_consume + callback
✅ RabbitTemplate 自动管理   ⚠️ 手动管理连接/信道
✅ 长连接常驻内存            ⚠️ PHP-FPM 需每次重连
✅ 线程池消费                ⚠️ 需 Swoole/Workerman 常驻

PHP 做消息队列的正确姿势：
- 消费端：Swoole 协程常驻 + 连接池
- 生产端：确认模式 + 失败落库重试
- 运维：Supervisor 守护 + 监控告警
```

---

## 13. 实战场景

### 13.1 下单异步链路

```php
final class OrderService
{
    public function create(array $orderData): array
    {
        // 1. 同步：写订单主表
        $orderId = $this->saveOrder($orderData);

        // 2. 异步：都通过消息队列
        $publisher = new ReliablePublisher();

        // 减库存
        $publisher->publish('order_events', 'stock.deduct', [
            'order_id' => $orderId,
            'skus'     => $orderData['skus'],
        ]);

        // 发短信
        $publisher->publish('order_events', 'sms.create_order', [
            'order_id' => $orderId,
            'phone'    => $orderData['phone'],
        ]);

        // 写用户行为日志
        $publisher->publish('order_events', 'log.behavior', [
            'user_id'   => $orderData['user_id'],
            'action'    => 'create_order',
            'order_id'  => $orderId,
        ]);

        return ['code' => 0, 'order_id' => $orderId];
    }
}
```

### 13.2 秒杀异步下单

```php
// ═══ 秒杀接口 ═══
final class SeckillController
{
    public function seckill(int $userId, int $productId): array
    {
        // 1. Redis 原子扣减
        $redis = new Redis();
        $redis->connect('127.0.0.1', 6379);
        $stock = $redis->decr("seckill:stock:{$productId}");

        if ($stock < 0) {
            $redis->incr("seckill:stock:{$productId}");
            return ['code' => -1, 'msg' => '已售罄'];
        }

        // 2. 防重复（用户已抢过）
        if (!$redis->set("seckill:user:{$userId}:{$productId}", 1, ['nx', 'ex' => 300])) {
            $redis->incr("seckill:stock:{$productId}");
            return ['code' => -2, 'msg' => '已参与'];
        }

        // 3. 投递到 MQ 异步下单
        $publisher = new ReliablePublisher();
        $publisher->publish('seckill', 'create_order', [
            'user_id'    => $userId,
            'product_id' => $productId,
            'timestamp'  => time(),
        ]);

        return ['code' => 0, 'msg' => '抢购成功，订单处理中'];
    }
}

// ═══ 秒杀订单消费者 ═══
$channel = RabbitMQFactory::getChannel();
$channel->queue_declare('seckill_orders', false, true, false, false);
$channel->basic_qos(null, 100, null);  // 一次取100条

$db = new PDO('mysql:host=127.0.0.1;dbname=mydb', 'root', 'pass');

$channel->basic_consume('seckill_orders', '', false, false, false, false,
    function ($msg) use ($db) {
        $data = json_decode($msg->body, true);

        try {
            // 批量写 MySQL（攒批）
            static $batch = [];
            $batch[] = [
                'user_id'    => $data['user_id'],
                'product_id' => $data['product_id'],
            ];

            if (count($batch) >= 50) {
                $db->beginTransaction();
                $stmt = $db->prepare('INSERT INTO orders (user_id, product_id, status) VALUES (?, ?, "paid")');
                foreach ($batch as $row) {
                    $stmt->execute([$row['user_id'], $row['product_id']]);
                }
                $db->commit();
                $batch = [];
            }

            $msg->ack();
        } catch (\Throwable $e) {
            error_log("下单失败: {$e->getMessage()}");
            $msg->nack(false);  // 进入死信
        }
    }
);
```

### 13.3 数据同步（Canal + MQ）

```
MySQL Binlog → Canal → RabbitMQ → PHP 消费者 → 更新 ES/Redis/Cache

架构：
  MySQL ───→ Canal ───→ MQ ───→ 索引更新/缓存刷新/数据清洗
```

---

## 14. 监控与运维

### 14.1 关键指标

```bash
# ═══ HTTP API 查看 ═══
# 队列状态
curl -u admin:admin123 http://localhost:15672/api/queues

# 连接数
curl -u admin:admin123 http://localhost:15672/api/connections

# 节点状态
curl -u admin:admin123 http://localhost:15672/api/nodes

# ═══ 命令行 ═══
rabbitmqctl list_queues name messages messages_ready messages_unacknowledged
rabbitmqctl list_exchanges
rabbitmqctl list_bindings
rabbitmqctl status

# ═══ 监控指标 ═══
# 队列堆积（messages_ready 持续增长 → 消费者跟不上）
# 未确��消息（messages_unacknowledged 大 → 消费者处理慢）
# 连接数增长 → 生产端没复用连接
# 内存/磁盘告警 → 触发 flow control（生产被限速）
```

### 14.2 监控脚本（PHP）

```php
final class RabbitMQMonitor
{
    private string $apiUrl = 'http://localhost:15672/api';

    public function checkQueueHealth(string $vhost, string $queue): array
    {
        $ch = curl_init("{$this->apiUrl}/queues/{$vhost}/{$queue}");
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_USERPWD        => 'admin:admin123',
            CURLOPT_TIMEOUT        => 5,
        ]);
        $data = json_decode(curl_exec($ch), true);

        return [
            'name'              => $data['name'],
            'messages_ready'    => $data['messages_ready'],
            'messages_unacked'  => $data['messages_unacknowledged'],
            'consumers'         => $data['consumers'],
            'rate_publish'      => $data['message_stats']['publish_details']['rate'] ?? 0,
            'rate_deliver'      => $data['message_stats']['deliver_get_details']['rate'] ?? 0,
        ];
    }

    public function shouldAlert(array $health): bool
    {
        // 堆积超过 10000 条告警
        if ($health['messages_ready'] > 10000) {
            return true;
        }

        // 没有消费者
        if ($health['consumers'] === 0) {
            return true;
        }

        // 消费速度显著落后生产速度
        if ($health['rate_publish'] > ($health['rate_deliver'] * 3)) {
            return true;
        }

        return false;
    }
}

// 定时执行（cron 每分钟）
$monitor = new RabbitMQMonitor();
$health = $monitor->checkQueueHealth('/', 'orders');
if ($monitor->shouldAlert($health)) {
    sendAlert("MQ 告警: orders 队列堆积 {$health['messages_ready']} 条");
}
```

---

---

## 15. RocketMQ 实战

### 15.1 RocketMQ 是什么

```
RocketMQ 是阿里开源的分布式消息中间件，双11核心基础设施。

架构：
┌──────────────────────────────────────────────┐
│               NameServer 集群                 │
│         （路由中心，类似注册中心）              │
└──────┬───────────────────────┬───────────────┘
       │                       │
┌──────▼──────┐         ┌──────▼──────┐
│  Broker-A    │         │  Broker-B    │
│  Master      │←──同步──│  Slave       │
│  ┌─────────┐│         │  ┌─────────┐│
│  │CommitLog││         │  │CommitLog││
│  │ConsumeQ ││         │  │ConsumeQ ││
│  └─────────┘│         │  └─────────┘│
└─────────────┘         └─────────────┘
       ▲                       ▲
       │        Producer        │
       │        Consumer        │

核心概念：
- NameServer：路由注册中心（无状态，集群部署）
- Broker：消息存储转发（Master-Slave 架构）
- Topic：主题（逻辑分类）
- Tag：子主题（消息过滤用）
- ConsumerGroup：消费组（同组内竞争消费）
- MessageQueue：队列（按数量分区）

RocketMQ 特色：
1. 事务消息（分布式事务最终一致性）
2. 顺序消息（全局有序 / 分区有序）
3. 18级延迟消息（1s 1s 1s 1s 1s 1s 1s 1s 1s 1s 2s 2s … 2h）
4. 消息过滤（Tag + SQL92 表达式）
5. 消息轨迹（追踪消息从生产到消费全链路）
6. 定时消息（精确到秒级）
```

### 15.2 PHP 安装与连接

```bash
# Docker 部署 RocketMQ
# docker-compose 启动 NameServer + Broker
docker run -d --name rmq-namesrv \
  -p 9876:9876 \
  apache/rocketmq:5.1.0 sh mqnamesrv

docker run -d --name rmq-broker \
  -p 10911:10911 -p 10909:10909 \
  -e "NAMESRV_ADDR=host.docker.internal:9876" \
  apache/rocketmq:5.1.0 sh mqbroker -n host.docker.internal:9876

# PHP 客户端
composer require aliyuncs/rocketmq-client-php
```

```php
use RocketMQ\Client\Producer;
use RocketMQ\Common\Message\Message;

// ═══ 生产者 ═══
$producer = new Producer('OrderProducerGroup');
$producer->setNamesrvAddr('127.0.0.1:9876');
$producer->start();

$msg = new Message('OrderTopic', 'create', 'KEY_001', json_encode([
    'order_id' => 1001,
    'user_id'  => 123,
]));

// 发送
$result = $producer->send($msg);
echo "发送成功: {$result->getMsgId()}\n";
$producer->shutdown();
```

### 15.3 事务消息（分布式事务）

```php
/**
 * 事务消息 = 确保本地事务 + 消息发送的原子性
 *
 * 流程：
 *   1. 发送 half 消息（半消息，消费者不可见）
 *   2. 执行本地事务
 *   3. 根据本地事务结果 commit 或 rollback
 *   4. 如果生产者挂了，Broker 回调 check 接口
 */

// RocketMQ 原生 PHP SDK 不完整，生产建议用 Java/Go 桥接
// 或者用阿里云 RocketMQ 5.x HTTP 协议

// HTTP 协议版本（阿里云）
final class RocketMQTransactionPublisher
{
    private string $endpoint = 'https://rmq-instance.aliyuncs.com';

    public function sendTransactional(string $topic, array $body): void
    {
        // 发送 half 消息
        $msgId = $this->sendHalfMessage($topic, $body);

        // 执行本地事务
        try {
            $this->executeLocalTransaction($body);
            $this->commit($topic, $msgId);
        } catch (\Throwable $e) {
            $this->rollback($topic, $msgId);
            throw $e;
        }
    }

    private function sendHalfMessage(string $topic, array $body): string
    {
        // POST /topics/{topic}/messages?transitType=HALF
        $ch = curl_init("{$this->endpoint}/topics/{$topic}/messages?transitType=HALF");
        curl_setopt_array($ch, [
            CURLOPT_POST           => true,
            CURLOPT_POSTFIELDS     => json_encode($body),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER     => ['Content-Type: application/json'],
        ]);
        $result = json_decode(curl_exec($ch), true);
        return $result['messageId'];
    }

    private function executeLocalTransaction(array $body): void
    {
        // 本地 DB 操作
        $db = new PDO('mysql:host=127.0.0.1;dbname=mydb', 'root', 'pass');
        $db->beginTransaction();
        $stmt = $db->prepare('INSERT INTO orders (...) VALUES (...)');
        $stmt->execute([$body['order_id']]);
        $db->commit();
    }

    private function commit(string $topic, string $msgId): void { /* ... */ }
    private function rollback(string $topic, string $msgId): void { /* ... */ }
}
```

### 15.4 RocketMQ vs RabbitMQ 重点差异

```
场景对比：

订单创建后的库存扣减：
  用 RocketMQ 事务消息 → 保证订单创建与消息发送原子性
  用 RabbitMQ + 本地消息表 → 额外维护消息表 + 定时扫描

顺序消息（同一订单的事件必须有序）：
  RocketMQ: send(msg, MessageQueueSelector) → 同订单同队列
  RabbitMQ: 单个 Queue 是有序的，但无原生 Hash 路由

延迟消息：
  RocketMQ: message.setDelayTimeLevel(3) → 18 个预定义级别
  RabbitMQ: TTL + DLX 方式，需额外创建延时队列

消息堆积：
  RocketMQ: 10 亿级堆积（CommitLog 顺序写）
  RabbitMQ: 百万级后性能下降（每个 Queue 独立文件）

海量消费：
  RocketMQ: 天然支持 Pull，消费者自主控制速率
  RabbitMQ: Push 为主，需要 basic_qos 限制预取
```

---

## 16. Kafka 实战

### 16.1 Kafka 是什么

```
Kafka 是 LinkedIn 开源的分布式流平台，最初用于日志收集。

架构：
┌─────────────────────────────────────────────────────┐
│                  Zookeeper / KRaft                   │
│               （元数据管理 / 控制器选举）              │
└──────┬──────────────────────┬───────────────────────┘
       │                      │
┌──────▼──────┐        ┌──────▼──────┐
│  Broker 1   │        │  Broker 2   │
│  ┌────────┐ │        │  ┌────────┐ │
│  │Topic-A │ │        │  │Topic-A │ │
│  │Part.0  │◄├────────┤  │Part.0  │ │ ← Replica
│  │Part.1  │ │        │  │Part.1  │◄├── Replica
│  │Topic-B │ │        │  │Topic-B │ │
│  │Part.0  │ │        │  │Part.0  │◄├── Replica
│  └────────┘ │        │  └────────┘ │
└─────────────┘        └─────────────┘
       ▲                      ▲
       │       Producer        │
       │       Consumer        │

核心概念：
- Broker：Kafka 服务节点
- Topic：主题（逻辑分类，类似 DB 的表）
- Partition：分区（物理存储单元，有序、不可变的消息序列）
- Offset：消息在分区中的唯一序号（单调递增）
- Producer：生产者，决定消息写入哪个分区
- ConsumerGroup：消费者组（组内分摊消费分区）
- Replica：副本（Leader 读写 / Follower 同步备份）

Kafka 为什么快：
1. 顺序写磁盘（随机写 100KB/s → 顺序写 600MB/s）
2. Page Cache（写入 OS 页缓存，批量落盘）
3. 零拷贝（sendfile 系统调用，不经过用户态）
4. 批量压缩发送
5. 分区并行 + ConsumerGroup 并行消费
```

### 16.2 PHP 安装与连接

```bash
# 安装 librdkafka（C 扩展）
sudo apt-get install -y librdkafka-dev
pecl install rdkafka

# 或 Docker
docker run -d --name kafka -p 9092:9092 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  apache/kafka:latest

# 高阶包（可选）
composer require nmred/kafka-php
composer require arnaud-lb/php-rdkafka
```

```php
// ═══ 生产者（使用 rdkafka 扩展） ═══
$conf = new RdKafka\Conf();
$conf->set('metadata.broker.list', 'localhost:9092');
$conf->set('acks', 'all');                    // 等所有副本确认
$conf->set('compression.codec', 'snappy');     // 压缩
$conf->set('batch.size', 16384);              // 批量大小

$producer = new RdKafka\Producer($conf);
$topic = $producer->newTopic('user_events');

// 发送消息（可指定分区 key）
$topic->produce(RD_KAFKA_PARTITION_UA, 0, json_encode([
    'user_id' => 123,
    'action'  => 'login',
    'time'    => date('Y-m-d H:i:s'),
]), 'user_123');  // key 决定分区

$producer->flush(10000);  // 等待发送完成

// ═══ 消费者 ═══
$conf = new RdKafka\Conf();
$conf->set('metadata.broker.list', 'localhost:9092');
$conf->set('group.id', 'log_consumer_group');
$conf->set('auto.offset.reset', 'earliest');  // 从头消费
$conf->set('enable.auto.commit', 'false');    // 手动提交 offset

$consumer = new RdKafka\KafkaConsumer($conf);
$consumer->subscribe(['user_events']);

while (true) {
    $message = $consumer->consume(1000);       // 等待 1 秒

    switch ($message->err) {
        case RD_KAFKA_RESP_ERR_NO_ERROR:
            $data = json_decode($message->payload, true);
            echo "收到: partition={$message->partition} offset={$message->offset}\n";
            processEvent($data);
            $consumer->commit($message);       // 手动提交
            break;

        case RD_KAFKA_RESP_ERR__PARTITION_EOF:
            echo "分区消费完毕\n";
            break;

        case RD_KAFKA_RESP_ERR__TIMED_OUT:
            break;

        default:
            echo "错误: {$message->errstr()}\n";
    }
}
```

### 16.3 Kafka 典型场景

```php
// ═══ 场景 1：用户行为日志管道 ═══
final class UserLogProducer
{
    private RdKafka\Producer $producer;

    public function __construct()
    {
        $conf = new RdKafka\Conf();
        $conf->set('metadata.broker.list', 'kafka:9092');
        $this->producer = new RdKafka\Producer($conf);
    }

    public function log(string $userId, string $action, array $context = []): void
    {
        $topic = $this->producer->newTopic('user_log');
        $topic->produce(RD_KAFKA_PARTITION_UA, 0, json_encode([
            'user_id'   => $userId,
            'action'    => $action,
            'context'   => $context,
            'timestamp' => (int)(microtime(true) * 1000),
        ]), $userId);  // 同一用户进同一分区（保序）
    }
}

// ═══ 场景 2：数据库 CDC（Change Data Capture） ═══
// MySQL Binlog → Kafka → 消费者更新 ES/Redis/缓存
// 常用工具：Debezium / Canal

$consumer->subscribe(['mysql_changes.mydb.users']);
$message = $consumer->consume(1000);
$change = json_decode($message->payload, true);
echo match ($change['op']) {
    'c' => "新增: id={$change['after']['id']}",
    'u' => "更新: id={$change['after']['id']}",
    'd' => "删除: id={$change['before']['id']}",
};
// 同步到 ES
$es->index([...$change['after']]);

// ═══ 场景 3：实时数据统计（Stream Processing） ═══
// Kafka → Flink / Spark Streaming → 实时大屏
// PHP 消费分区取实时数据
$partition = $message->partition;
$count = $redis->incr("realtime:pv:partition:{$partition}");
```

### 16.4 四大家族终极对比

```
                        RabbitMQ      RocketMQ      Kafka         Redis Stream
                        ────────      ────────      ─────         ────────────
定位：                  企业消息总线   电商交易引擎   流处理平台    轻量消息
吞吐（单机）：           万级/秒        十万级/秒      百万级/秒    十万级/秒
延迟：                  微秒级         毫秒级         毫秒级       微秒级
可靠程度：              ★★★★★         ★★★★★         ★★★★★        ★★★
事务消息：              ❌             ✅             ❌           ❌
原生延迟：              插件           18级           无           无
顺序消息：              Queue有序      原生支持        分区有序      ❌
消息回溯：              ❌             ✅             ✅           ❌
多语言SDK：             极多           一般           多           Redis客户端
PHP SDK成熟度：         ★★★★★         ★★            ★★★★        ★★★★★
运维成本：              中低           高             高           低

一句话选型：
- 复杂路由/企业集成 → RabbitMQ
- 电商交易/事务消息 → RocketMQ
- 大数据流/日志管道 → Kafka
- 已有Redis/轻量异步 → Redis Stream
```

> 🐇 **消息队列是分布式系统的基石。掌握 RabbitMQ 的交换机/死信/ACK/延迟，理解 RocketMQ 的事务消息/顺序消息和 Kafka 的高吞吐/流处理，PHP 也能驾驭复杂的异步架构。**
