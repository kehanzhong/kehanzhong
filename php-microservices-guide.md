# PHP 微服务架构实战：从拆分到治理

> 服务拆分、通信协议、API 网关、服务发现、分布式事务、监控 — 全链路实战

---

## 目录

1. [微服务核心概念与拆分策略](#1-微服务核心概念与拆分策略)
2. [服务间通信：REST、gRPC、消息队列](#2-服务间通信restgrpc消息队列)
3. [API 网关：Kong & Traefik](#3-api-网关kong--traefik)
4. [服务发现与配置中心：Consul](#4-服务发现与配置中心consul)
5. [分布式事务：Saga 模式](#5-分布式事务saga-模式)
6. [容器化编排与部署](#6-容器化编排与部署)
7. [可观测性：日志、链路追踪、指标](#7-可观测性日志链路追踪指标)
8. [完整电商微服务拆解实战](#8-完整电商微服务拆解实战)
9. [避坑指南与设计原则](#9-避坑指南与设计原则)

---

## 1. 微服务核心概念与拆分策略

### 1.1 何时该上微服务？

```
不用微服务的信号：
  ✅ 项目 < 3 人
  ✅ 月活 < 10 万
  ✅ 一个 DB 够用
  ✅ 发布频率 < 每周 1 次

考虑微服务的信号：
  ⚠️ 不同模块有截然不同的流量特征（如搜索 vs 支付）
  ⚠️ 部分模块需要独立扩缩容
  ⚠️ 一个团队改代码总被另一个团队阻塞
  ⚠️ 部署时即使改一行代码也要全量回归
```

### 1.2 拆分原则

| 原则 | 说明 | 反例 |
|------|------|------|
| **单一职责** | 一个服务只做一件事 | 用户服务里写订单逻辑 |
| **按业务域** | 按 DDD 限界上下文拆分 | 按技术层拆分（Controller层/Service层） |
| **数据独立** | 每个服务有自己的数据库 | 多个服务共享一个 DB |
| **接口先行** | 服务间只通过 API 通信 | 直接连对方数据库查数据 |
| **独立部署** | 每个服务独立构建、测试、发布 | 所有服务一起发版 |
| **故障隔离** | 一个服务挂了不影响其他 | 同步调用链雪崩 |

### 1.3 电商系统拆分示例

```
单体应用 (monolith)
  └── app/
      ├── User/            # 用户模块
      ├── Product/         # 商品模块
      ├── Order/           # 订单模块
      ├── Payment/         # 支付模块
      ├── Inventory/       # 库存模块
      └── Notification/    # 通知模块

微服务拆分后
  ├── user-service         # 用户服务 (DB: user_db)
  ├── product-service      # 商品服务 (DB: product_db)
  ├── order-service        # 订单服务 (DB: order_db)
  ├── payment-service      # 支付服务 (DB: payment_db)
  ├── inventory-service    # 库存服务 (DB: inventory_db)
  └── notification-service # 通知服务 (无主 DB，只消费消息)
```

### 1.4 单体到微服务迁移路线

```
单体
  → 拆分数据（表按业务域分组）
    → 抽取模块（独立 composer 包或私有 Git 仓库）
      → 独立部署（独立容器，API 通信）
        → 独立数据库（逐步迁移表）

不要在第一步就拆 DB！先拆代码，再拆数据。
```

---

## 2. 服务间通信：REST、gRPC、消息队列

### 2.1 REST + JSON（同步通信）

**适用：** 查询类请求、实时性要求不高的写入

```php
// 订单服务调用用户服务获取用户信息
class UserServiceClient
{
    public function __construct(
        private \GuzzleHttp\Client $http,
        private string $baseUrl = 'http://user-service',
    ) {}

    public function getUser(int $userId): ?array
    {
        try {
            $response = $this->http->get("{$this->baseUrl}/api/users/{$userId}", [
                'headers' => [
                    'X-Request-ID' => $this->getTraceId(),
                    'Accept'       => 'application/json',
                ],
                'timeout' => 3,
            ]);

            return json_decode($response->getBody(), true);
        } catch (\GuzzleHttp\Exception\ConnectException $e) {
            // 熔断：返回缓存值或降级
            return $this->getCachedUser($userId);
        } catch (\GuzzleHttp\Exception\RequestException $e) {
            throw new ServiceUnavailableException(
                "user-service 不可用: {$e->getMessage()}"
            );
        }
    }

    private function getTraceId(): string
    {
        return $_SERVER['HTTP_X_TRACE_ID'] ?? bin2hex(random_bytes(16));
    }
}
```

### 2.2 gRPC（高性能同步通信）

```protobuf
// proto/user.proto
syntax = "proto3";
package user;

service UserService {
    rpc GetUser (GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest {
    int32 user_id = 1;
}

message GetUserResponse {
    int32 id = 1;
    string name = 2;
    string email = 3;
}
```

```bash
# 生成 PHP 代码
composer require grpc/grpc protobuf-php/protobuf
protoc --php_out=./generated --grpc_out=./generated proto/user.proto
```

```php
// PHP gRPC 客户端
$client = new User\UserServiceClient(
    'user-service:50051',
    ['credentials' => \Grpc\ChannelCredentials::createInsecure()]
);

$request = new User\GetUserRequest();
$request->setUserId(42);

list($response, $status) = $client->GetUser($request)->wait();
if ($status->code === \Grpc\STATUS_OK) {
    echo $response->getName(); // 用户名
}
```

**REST vs gRPC 选型：**

| | REST + JSON | gRPC |
|---|---|---|
| 可读性 | ✅ 人类可读 | ❌ 二进制 |
| 性能 | ⭐⭐ | ⭐⭐⭐ |
| 强类型 | ❌ | ✅ |
| 浏览器支持 | ✅ | ❌（需 grpc-web） |
| 适用 | 外部 API / 简单服务 | 内部高性能服务间调用 |

### 2.3 消息队列（异步通信）

**适用：** 写入类请求、事件广播、削峰填谷

```php
// 订单服务发布事件
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

class OrderEventPublisher
{
    public function __construct(private AMQPStreamConnection $connection) {}

    public function orderCreated(array $order): void
    {
        $channel = $this->connection->channel();
        $channel->exchange_declare('order', 'topic', false, true, false);

        $message = new AMQPMessage(json_encode([
            'event'     => 'order.created',
            'timestamp' => time(),
            'data'      => $order,
        ]), ['delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT]);

        $channel->basic_publish($message, 'order', 'order.created');
    }
}

// 库存服务消费事件
class OrderCreatedListener
{
    public function handle(string $body): void
    {
        $event = json_decode($body, true);
        $order = $event['data'];

        // 扣减库存
        foreach ($order['items'] as $item) {
            $this->inventoryService->deduct($item['product_id'], $item['quantity']);
        }

        // 发布"库存已扣减"事件
        $this->publisher->inventoryDeducted($order['id']);
    }
}
```

**通信模式选择决策树：**

```
需要立即返回结果？
  ├── 是 → 同步通信
  │   ├── 内部高性能服务间 → gRPC
  │   └── 对外 API / 简单场景 → REST
  └── 否 → 异步通信
      ├── 事件广播 → 消息队列 (pub/sub)
      └── 命令/任务 → 消息队列 (work queue)
```

---

## 3. API 网关：Kong & Traefik

### 3.1 为什么需要网关？

```
没有网关：
  客户端 → user-service:8001
  客户端 → product-service:8002
  客户端 → order-service:8003
  ❌ 多域名/端口，跨域问题，认证散落各处

有网关：
  客户端 → https://api.example.com (网关)
    ├── /users/*    → user-service
    ├── /products/* → product-service
    └── /orders/*   → order-service
  ✅ 统一入口，认证/限流/日志集中在网关
```

### 3.2 Kong + Docker Compose

```yaml
# docker-compose.yml
version: '3.8'
services:
  kong-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: kong
      POSTGRES_USER: kong
      POSTGRES_PASSWORD: kong_pass
    volumes:
      - kong_db:/var/lib/postgresql/data

  kong-migration:
    image: kong:3.6
    command: kong migrations bootstrap
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-db
      KONG_PG_PASSWORD: kong_pass
    depends_on:
      - kong-db

  kong:
    image: kong:3.6
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-db
      KONG_PG_PASSWORD: kong_pass
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
      KONG_PROXY_LISTEN: 0.0.0.0:8000
    ports:
      - "8000:8000"   # 网关代理端口
      - "8001:8001"   # 管理 API
      - "8443:8443"
    depends_on:
      - kong-migration

volumes:
  kong_db:
```

### 3.3 Kong 路由配置

```bash
# 添加服务
curl -X POST http://localhost:8001/services \
  -d name=user-service \
  -d url=http://host.docker.internal:8101

# 添加路由
curl -X POST http://localhost:8001/services/user-service/routes \
  -d name=user-route \
  -d paths[]=/users \
  -d strip_path=false

# 启用 JWT 认证插件
curl -X POST http://localhost:8001/services/user-service/plugins \
  -d name=jwt

# 启用限流插件（每秒 100 次）
curl -X POST http://localhost:8001/services/user-service/plugins \
  -d name=rate-limiting \
  -d config.second=100

# 启用 CORS
curl -X POST http://localhost:8001/services/user-service/plugins \
  -d name=cors \
  -d config.origins=*
```

```bash
# 访问网关（自动路由到 user-service）
curl http://localhost:8000/users/api/users/1

# 带 JWT 的请求
curl http://localhost:8000/users/api/users/1 \
  -H "Authorization: Bearer eyJhbG..."
```

### 3.4 Traefik（轻量替代）

```yaml
# docker-compose.yml
services:
  traefik:
    image: traefik:v3.1
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"   # Dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  user-service:
    image: myapp/user-service
    labels:
      - "traefik.http.routers.user.rule=Host(`api.example.com`) && PathPrefix(`/users`)"
      - "traefik.http.middlewares.user-strip.stripprefix.prefixes=/users"
      - "traefik.http.routers.user.middlewares=user-strip"

  product-service:
    image: myapp/product-service
    labels:
      - "traefik.http.routers.product.rule=Host(`api.example.com`) && PathPrefix(`/products`)"
      - "traefik.http.routers.product.middlewares=product-strip"
```

**Kong vs Traefik：**

| | Kong | Traefik |
|---|---|---|
| 插件生态 | ⭐⭐⭐ 丰富（JWT/限流/日志/认证） | ⭐⭐ 中等 |
| 配置方式 | REST API + 声明式 | 标签 + 文件/Consul |
| 性能 | ⭐⭐ | ⭐⭐⭐（Go） |
| 学习曲线 | ⭐⭐ 中 | ⭐ 低 |
| 适合场景 | API 管理重度需求 | Docker/K8s 原生部署 |

---

## 4. 服务发现与配置中心：Consul

### 4.1 Consul 三合一

```
Consul 提供三大能力：
  1. 服务发现    — 服务注册、健康检查、DNS/HTTP 发现
  2. 配置中心    — KV 存储，动态配置热更新
  3. 健康检查    — 自动摘除不健康节点
```

```yaml
# docker-compose.yml
services:
  consul:
    image: hashicorp/consul:1.19
    ports:
      - "8500:8500"
      - "8600:8600/udp"
    command: agent -server -bootstrap -ui -client=0.0.0.0
```

### 4.2 PHP 服务注册

```php
use SensioLabs\Consul\ServiceFactory;
use SensioLabs\Consul\Services\AgentInterface;

class ServiceRegistry
{
    private AgentInterface $agent;

    public function __construct()
    {
        $factory = new ServiceFactory(['base_uri' => 'http://consul:8500']);
        $this->agent = $factory->get(AgentInterface::class);
    }

    public function register(string $name, int $port, string $healthUrl = '/health'): void
    {
        $this->agent->registerService([
            'ID'      => "{$name}-" . gethostname(),
            'Name'    => $name,
            'Port'    => $port,
            'Address' => $this->getLocalIp(),
            'Tags'    => ['php', 'v1'],
            'Check'   => [
                'HTTP'     => "http://{$this->getLocalIp()}:{$port}{$healthUrl}",
                'Interval' => '10s',
                'Timeout'  => '3s',
                'DeregisterCriticalServiceAfter' => '60s',
            ],
        ]);
    }

    private function getLocalIp(): string
    {
        return gethostbyname(gethostname());
    }
}

// 启动时注册，退出时注销
$registry = new ServiceRegistry();
$registry->register('user-service', 8101);

pcntl_signal(SIGTERM, function() use ($registry) {
    $registry->agent->deregisterService("user-service-" . gethostname());
    exit;
});
```

### 4.3 PHP 服务发现

```php
use SensioLabs\Consul\Services\HealthInterface;

class ServiceDiscovery
{
    private HealthInterface $health;

    public function __construct()
    {
        $factory = new ServiceFactory(['base_uri' => 'http://consul:8500']);
        $this->health = $factory->get(HealthInterface::class);
    }

    public function resolve(string $serviceName): string
    {
        $services = $this->health->service($serviceName, [
            'passing' => true,  // 只返回健康实例
        ]);

        $instances = json_decode($services->getBody(), true);
        if (empty($instances)) {
            throw new \RuntimeException("无可用实例: {$serviceName}");
        }

        // 随机选一个（简单负载均衡）
        $instance = $instances[array_rand($instances)];
        return "http://{$instance['Service']['Address']}:{$instance['Service']['Port']}";
    }
}

// 使用
$discovery = new ServiceDiscovery();
$userServiceUrl = $discovery->resolve('user-service');

$response = (new \GuzzleHttp\Client())->get("{$userServiceUrl}/api/users/1");
```

### 4.4 配置中心使用

```php
// 从 Consul KV 读取配置
use SensioLabs\Consul\Services\KVInterface;

class ConsulConfig
{
    private KVInterface $kv;

    public function __construct()
    {
        $factory = new ServiceFactory(['base_uri' => 'http://consul:8500']);
        $this->kv = $factory->get(KVInterface::class);
    }

    public function get(string $key, mixed $default = null): mixed
    {
        try {
            $value = $this->kv->get("config/{$key}");
            $raw = json_decode($value->getBody(), true);
            return base64_decode($raw[0]['Value'] ?? '');
        } catch (\Exception $e) {
            return $default;
        }
    }
}

// services 配置写进 Consul KV
// consul kv put config/services/sms/gateway aliyun
// consul kv put config/services/sms/limit 1000
```

```bash
# Consul KV 操作
consul kv put config/services/payment/stripe/key sk_live_xxx
consul kv get config/services/payment/stripe/key
consul kv delete config/services/payment/oldkey
```

---

## 5. 分布式事务：Saga 模式

### 5.1 问题场景

```
创建订单流程（跨三个服务）：
  order-service    创建订单 "pending"
  inventory-service  扣减库存
  payment-service    发起扣款

如果第 3 步失败，需要回滚第 2 步（恢复库存）和第 1 步（取消订单）
```

### 5.2 编排式 Saga（推荐）

```php
// 订单服务 —Saga 编排器
class CreateOrderSaga
{
    private array $steps = [];
    private array $compensations = [];

    public function addStep(callable $forward, callable $compensation): void
    {
        $this->steps[] = $forward;
        $this->compensations[] = $compensation;
    }

    public function execute(): void
    {
        $completed = [];

        try {
            for ($i = 0; $i < count($this->steps); $i++) {
                $this->steps[$i]();
                $completed[] = $i; // 记录已完成的步骤
            }
        } catch (\Throwable $e) {
            // 逆序执行补偿
            for ($i = count($completed) - 1; $i >= 0; $i--) {
                $this->compensations[$i]();
            }
            throw $e;
        }
    }
}

// 实际使用
$orderId = bin2hex(random_bytes(16));

$saga = new CreateOrderSaga();

// 步骤 1：创建订单
$saga->addStep(
    fn() => $this->orderRepo->create(['id' => $orderId, 'status' => 'pending']),
    fn() => $this->orderRepo->updateStatus($orderId, 'cancelled'),
);

// 步骤 2：扣减库存
$saga->addStep(
    function() use ($order) {
        $res = (new GuzzleHttp\Client())->post('http://inventory-service/api/inventory/deduct', [
            'json' => ['items' => $order['items']],
        ]);
        if ($res->getStatusCode() !== 200) {
            throw new \Exception('库存不足');
        }
    },
    fn() => (new GuzzleHttp\Client())->post('http://inventory-service/api/inventory/restore', [
        'json' => ['order_id' => $orderId],
    ]),
);

// 步骤 3：扣款
$saga->addStep(
    fn() => $this->paymentService->charge($orderId, $order['total']),
    fn() => $this->paymentService->refund($orderId),
);

$saga->execute();
$this->orderRepo->updateStatus($orderId, 'paid');
```

### 5.3 协同式 Saga（事件驱动）

```
order-service   发布 "OrderCreated" 事件
  → inventory-service 消费 → 扣库存 → 发布 "InventoryDeducted"
    → payment-service 消费 → 扣款 → 发布 "PaymentCharged"
      → order-service 消费 → 状态改为 "completed"

如果 payment-service 扣款失败：
  → 发布 "PaymentFailed" 事件
    → inventory-service 消费 → 恢复库存
    → order-service 消费 → 状态改为 "cancelled"
```

### 5.4 最终一致性处理

```php
// 订单服务：补偿机制
class OrderEventHandler
{
    public function onPaymentFailed(array $event): void
    {
        $orderId = $event['data']['order_id'];

        // 取消订单
        $this->orderRepo->updateStatus($orderId, 'cancelled', [
            'reason' => $event['data']['reason'],
        ]);

        // 发送通知
        event(new OrderCancelled($orderId));
    }

    // 超过 30 分钟未支付 → 自动取消
    public function onTimeoutCheck(): void
    {
        $orders = Order::where('status', 'pending')
            ->where('created_at', '<', now()->subMinutes(30))
            ->get();

        foreach ($orders as $order) {
            $this->cancelOrder($order->id, '超时未支付');
        }
    }
}
```

---

## 6. 容器化编排与部署

### 6.1 PHP 微服务 Dockerfile

```dockerfile
FROM php:8.2-fpm-alpine

RUN apk add --no-cache \
    git unzip curl icu-dev libpq-dev \
    && docker-php-ext-install pdo pdo_mysql opcache intl bcmath

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/user-service
COPY . .

RUN composer install --no-dev --optimize-autoloader --prefer-dist && \
    chown -R www-data:www-data .

EXPOSE 9000
CMD ["php-fpm"]
```

### 6.2 Docker Compose 微服务编排

```yaml
# docker-compose.yml
version: '3.8'

services:
  # === 基础设施 ===
  consul:
    image: hashicorp/consul:1.19
    ports: ["8500:8500"]
    command: agent -server -bootstrap -ui -client=0.0.0.0

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    ports: ["5672:5672", "15672:15672"]
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin

  # === API 网关 ===
  kong:
    image: kong:3.6
    ports: ["8000:8000", "8001:8001"]
    environment:
      KONG_DATABASE: "off"
      KONG_DECLARATIVE_CONFIG: /kong.yml
    volumes:
      - ./kong.yml:/kong.yml

  # === 微服务 ===
  user-service:
    build: ./services/user
    environment:
      APP_ENV: production
    depends_on: [consul]

  product-service:
    build: ./services/product
    depends_on: [consul]

  order-service:
    build: ./services/order
    depends_on: [consul, rabbitmq]

  inventory-service:
    build: ./services/inventory
    depends_on: [consul, rabbitmq]

  # === 数据库（每个服务独立 DB）===
  user-db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: user_db

  product-db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: product_db

  order-db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: order_db
```

### 6.3 灰度发布

```yaml
# 金丝雀发布：v2 先接 10% 流量
user-service:
  image: user-service:v1
  deploy:
    replicas: 9

user-service-canary:
  image: user-service:v2
  deploy:
    replicas: 1
```

```bash
# 利用 Nginx 分流
# nginx.conf
upstream user_service {
    server user-service:8101   weight=90;  # v1 90%
    server user-service-v2:8102 weight=10;  # v2 10%
}
```

---

## 7. 可观测性：日志、链路追踪、指标

### 7.1 结构化日志

```php
// 每个微服务使用统一日志格式
class JsonLogger
{
    public function log(string $level, string $message, array $context = []): void
    {
        echo json_encode([
            'timestamp' => date('c'),
            'level'     => $level,
            'message'   => $message,
            'service'   => getenv('SERVICE_NAME'),
            'host'      => gethostname(),
            'trace_id'  => $context['trace_id'] ?? null,
            'span_id'   => $context['span_id'] ?? null,
            'context'   => $context,
        ]) . "\n";
    }
}

// 通过 Fluentd / Filebeat 收集到 ELK
```

### 7.2 链路追踪（OpenTelemetry）

```php
// 使用 OpenTelemetry PHP SDK
use OpenTelemetry\API\Trace\TracerInterface;

class TracedHttpClient
{
    public function __construct(
        private TracerInterface $tracer,
        private \GuzzleHttp\Client $http,
    ) {}

    public function call(string $service, string $method, string $path): array
    {
        $span = $this->tracer->spanBuilder("{$service}/{$path}")
            ->setAttribute('http.method', $method)
            ->setAttribute('http.url', $path)
            ->startSpan();

        try {
            $response = $this->http->request($method, $path, [
                'headers' => [
                    'X-Trace-Id' => $span->getContext()->getTraceId(),
                    'X-Span-Id'  => $span->getContext()->getSpanId(),
                ],
            ]);

            $span->setAttribute('http.status_code', $response->getStatusCode());
            return json_decode($response->getBody(), true);
        } finally {
            $span->end();
        }
    }
}

// 导出到 Jaeger / Zipkin / Grafana Tempo
```

### 7.3 指标监控

```php
// 自定义业务指标
class MetricsCollector
{
    private array $counters = [];
    private array $timers = [];

    public function increment(string $metric, int $value = 1): void
    {
        $this->counters[$metric] = ($this->counters[$metric] ?? 0) + $value;
    }

    public function timing(string $metric, float $ms): void
    {
        $this->timers[$metric][] = $ms;
    }

    // 暴露 /metrics 端点给 Prometheus 抓取
    public function render(): string
    {
        $out = '';
        foreach ($this->counters as $name => $value) {
            $out .= "{$name} {$value}\n";
        }
        foreach ($this->timers as $name => $values) {
            $avg = array_sum($values) / count($values);
            $out .= "{$name}_avg {$avg}\n";
        }
        return $out;
    }
}

// Prometheus 配置
// scrape_configs:
//   - job_name: 'php-microservices'
//     static_configs:
//       - targets: ['user-service:9090', 'order-service:9090']
```

### 7.4 监控仪表盘

| 工具 | 用途 |
|------|------|
| **Grafana** | 统一的监控仪表盘 |
| **Prometheus** | 指标收集 + 告警规则 |
| **Jaeger** | 分布式链路追踪 |
| **ELK Stack** | 日志聚合与搜索 |
| **Kibana** | 日志可视化 |

---

## 8. 完整电商微服务拆解实战

### 8.1 架构总览

```
                    ┌─────────────────────┐
                    │    API Gateway       │
                    │  (Kong / Traefik)    │
                    └──────┬──────────────┘
                           │
          ┌────────────────┼──────────────────┐
          │                │                   │
    ┌─────▼─────┐   ┌─────▼─────┐    ┌───────▼──────┐
    │  User     │   │ Product   │    │   Order      │
    │  Service  │   │ Service   │    │   Service    │
    │  (REST)   │   │ (REST)    │    │ (REST + MQ)  │
    └───────────┘   └───────────┘    └──────┬───────┘
                                            │
                    ┌───────────────────────┼──────────────┐
                    │                       │               │
             ┌──────▼──────┐   ┌───────────▼──┐   ┌───────▼──────┐
             │ Inventory   │   │  Payment     │   │ Notification │
             │ Service     │   │  Service     │   │   Service    │
             │ (MQ)        │   │ (MQ)         │   │   (MQ only)  │
             └─────────────┘   └──────────────┘   └──────────────┘
```

### 8.2 目录结构

```
microservices-ecommerce/
├── docker-compose.yml
├── kong.yml                    # 网关声明式配置
├── consul/                     # Consul 配置
├── services/
│   ├── user-service/
│   │   ├── Dockerfile
│   │   ├── composer.json
│   │   ├── app/
│   │   └── config/
│   ├── product-service/
│   ├── order-service/
│   │   ├── app/
│   │   │   ├── Saga/
│   │   │   │   └── CreateOrderSaga.php
│   │   │   └── Events/
│   │   └── ...
│   ├── inventory-service/
│   ├── payment-service/
│   └── notification-service/
├── shared/                     # 共享库
│   ├── ServiceDiscovery.php
│   ├── JsonLogger.php
│   └── SagaOrchestrator.php
└── proto/                      # gRPC 定义
    ├── user.proto
    └── order.proto
```

### 8.3 订单服务完整流程

```php
// order-service/app/Controllers/OrderController.php
class OrderController
{
    public function __construct(
        private ServiceDiscovery $discovery,
        private \GuzzleHttp\Client $http,
        private OrderEventPublisher $publisher,
    ) {}

    public function create(Request $request): JsonResponse
    {
        $data = $request->validate([
            'user_id' => 'required|int',
            'items'   => 'required|array',
        ]);

        // 1. 获取用户（校验存在性）
        $userServiceUrl = $this->discovery->resolve('user-service');
        $user = $this->http->get("{$userServiceUrl}/api/users/{$data['user_id']}");
        if ($user->getStatusCode() !== 200) {
            return response()->json(['error' => '用户不存在'], 404);
        }

        // 2. 获取商品价格
        $productServiceUrl = $this->discovery->resolve('product-service');
        $products = $this->http->post("{$productServiceUrl}/api/products/batch", [
            'json' => ['ids' => array_column($data['items'], 'product_id')],
        ]);

        // 3. 创建订单 Saga
        $orderId = bin2hex(random_bytes(16));
        $saga = new CreateOrderSaga();

        $saga->addStep(
            fn() => $this->orderRepo->create([
                'id'     => $orderId,
                'user_id'=> $data['user_id'],
                'items'  => $data['items'],
                'total'  => $this->calculateTotal($data['items'], json_decode($products->getBody(), true)),
                'status' => 'pending',
            ]),
            fn() => $this->orderRepo->updateStatus($orderId, 'cancelled'),
        );

        $saga->addStep(
            fn() => $this->http->post(
                $this->discovery->resolve('inventory-service') . '/api/inventory/deduct',
                ['json' => ['order_id' => $orderId, 'items' => $data['items']]],
            ),
            fn() => $this->http->post(
                $this->discovery->resolve('inventory-service') . '/api/inventory/restore',
                ['json' => ['order_id' => $orderId]],
            ),
        );

        // 4. 发布事件（异步扣款）
        $saga->addStep(
            fn() => $this->publisher->orderCreated(['id' => $orderId, 'total' => $total]),
            fn() => $this->publisher->orderCancelled($orderId),
        );

        $saga->execute();

        return response()->json([
            'order_id' => $orderId,
            'status'   => 'pending',
        ], 201);
    }
}
```

---

## 9. 避坑指南与设计原则

### 9.1 十大教训

| # | 教训 | 正确做法 |
|---|------|----------|
| 1 | 没想清楚就拆 | 先单体 MVP → 验证业务 → 再拆 |
| 2 | 共享数据库 | 每个服务独立 DB，只通过 API 读对方数据 |
| 3 | 同步链过长 | A→B→C→D 改成 MQ 异步 |
| 4 | 没有熔断 | 引入断路器（超时+降级） |
| 5 | 没有幂等 | 消费端检查 `message_id` 去重 |
| 6 | 没有链路追踪 | 每条请求带 `X-Trace-Id` |
| 7 | 手动部署 | Docker + CI/CD + 健康检查 |
| 8 | 没有监控 | Prometheus 指标 + Grafana 仪表盘 |
| 9 | 微服务改造成本低估 | 预留 30% 时间处理一致性/调试/排障 |
| 10 | 以为微服务更快 | 网络开销+序列化 通常比单体慢 10-20% |

### 9.2 设计原则

```
1. 高内聚、低耦合 — 一个服务改内部逻辑不应影响其他服务
2. 故障隔离 — 一个服务挂了，其他服务继续运行（熔断+降级）
3. Design for Failure — 假设网络会断、服务会超时、消息会丢失
4. 最终一致性 — 不强求强一致，用补偿替代回滚
5. API 版本化 — /v1/、/v2/，不要破坏性改接口
6. 基础设施即代码 — Dockerfile + docker-compose 版本化
```

### 9.3 单体 vs 微服务速查

| | 单体 | 微服务 |
|---|---|---|
| 开发速度（初期） | ⭐⭐⭐ 快 | ⭐ 慢 |
| 部署复杂度 | ⭐ 低 | ⭐⭐⭐ 高 |
| 调试排障 | ⭐ 简单 | ⭐⭐⭐ 复杂 |
| 独立扩缩 | ❌ | ✅ |
| 技术栈灵活 | ❌ | ✅ |
| 团队自治 | ❌ | ✅ |
| 故障隔离 | ❌ | ✅ |

---

> **文档版本**: 1.0  
> **更新日期**: 2026-05-14  
> **适用场景**: PHP 项目微服务架构设计、单体拆分与服务治理
