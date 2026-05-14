# RESTful API 设计规范

> 从 URL 设计到版本管理，前后端分离时代的标准接口指南

---

## 目录

1. [核心原则](#1-核心原则)
2. [URL 设计](#2-url-设计)
3. [HTTP 方法与状态码](#3-http-方法与状态码)
4. [参数设计](#4-参数设计)
5. [响应格式](#5-响应格式)
6. [错误码体系](#6-错误码体系)
7. [分页设计](#7-分页设计)
8. [版本管理](#8-版本管理)
9. [认证与授权](#9-认证与授权)
10. [接口文档](#10-接口文档)
11. [限流设计](#11-限流设计)
12. [PHP 完整实现](#12-php-完整实现)

---

## 1. 核心原则

```
REST 六大约束：
1. 客户端-服务器：分离关注点
2. 无状态：每个请求包含全部信息（不依赖服务器Session）
3. 可缓存：响应明确标记可否缓存
4. 分层系统：客户端不知道是否直连服务器
5. 统一接口：资源标识 / 操作语义 / 自描述 / HATEOAS
6. 按需代码：服务器可下发可执行代码（可选）

实践要点：
- URL 定位资源，HTTP 方法定义操作
- GET /users/123    → 查看用户（幂等）
- POST /users       → 创建用户
- PUT /users/123    → 全量更新（幂等）
- PATCH /users/123  → 部分更新
- DELETE /users/123 → 删除（幂等）
```

---

## 2. URL 设计

```php
// ═══ ✅ 好的设计 ═══

// 资源复数
GET    /api/v1/users             // 用户列表
GET    /api/v1/users/123         // 单个用户
POST   /api/v1/users             // 创建用户
PUT    /api/v1/users/123         // 更新用户
DELETE /api/v1/users/123         // 删除用户

// 子资源（不超过2层）
GET    /api/v1/users/123/orders          // 用户的订单
GET    /api/v1/users/123/orders/456      // 某用户的某个订单
POST   /api/v1/users/123/orders          // 为用户创建订单

// 非 CRUD 操作（动词命名）
POST   /api/v1/orders/123/cancel         // 取消订单
POST   /api/v1/users/123/reset-password  // 重置密码
POST   /api/v1/sms/verify-code           // 验证短信验证码

// 批量操作
POST   /api/v1/batch/users/delete        // 批量删除
PATCH  /api/v1/batch/orders/status       // 批量修改状态


// ═══ ❌ 坏的设计 ═══

GET    /getUserList              // 动词、不规范
POST   /deleteUser?id=123        // 用POST做删除
GET    /users/123/orders/comments/likes  // 太深
GET    /api/addUser              // 不区分版本
```

---

## 3. HTTP 方法与状态码

```php
// ═══ 状态码速查 ═══

2xx 成功
  200 OK               // 通用成功
  201 Created          // 创建成功（POST）
  204 No Content       // 删除成功 / 无返回内容
  206 Partial Content  // 分页 / 范围请求

3xx 重定向
  301 Moved Permanently  // 资源永久迁移
  304 Not Modified        // 缓存未变更（ETag/If-Modified-Since）

4xx 客户端错误
  400 Bad Request         // 参数错误
  401 Unauthorized        // 未认证（没登录/token过期）
  403 Forbidden           // 已认证但没权限
  404 Not Found           // 资源不存在
  405 Method Not Allowed  // 方法不支持
  409 Conflict            // 资源冲突（重复创建）
  422 Unprocessable Entity // 参数验证失败
  429 Too Many Requests   // 限流

5xx 服务端错误
  500 Internal Server Error  // 服务端异常
  502 Bad Gateway            // 网关错误
  503 Service Unavailable    // 服务不可用（维护中）
  504 Gateway Timeout        // 网关超时
```

---

## 4. 参数设计

```php
// ═══ 过滤 ═══
GET /api/v1/users?status=active&role=admin

// ═══ 排序 ═══
GET /api/v1/users?sort=-created_at,id    // +升序 -降序

// ═══ 字段选择 ═══
GET /api/v1/users?fields=id,name,avatar  // 只返回指定字段

// ═══ 搜索 ═══
GET /api/v1/users?q=张三

// ═══ 展开关联 ═══
GET /api/v1/orders?include=user,items.product

// ═══ 实用组合 ═══
GET /api/v1/orders?status=pending&sort=-created_at&page=1&per_page=20&include=user
```

---

## 5. 响应格式

### 5.1 统一响应结构

```json
// ═══ 成功（单个） ═══
{
  "code": 0,
  "message": "ok",
  "data": {
    "id": 123,
    "name": "张三",
    "email": "zhangsan@example.com",
    "created_at": "2026-01-15T10:30:00+08:00"
  }
}

// ═══ 成功（列表） ═══
{
  "code": 0,
  "message": "ok",
  "data": [
    { "id": 1, "name": "张三" },
    { "id": 2, "name": "李四" }
  ],
  "meta": {
    "current_page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  },
  "links": {
    "first": "/api/v1/users?page=1",
    "prev": null,
    "next": "/api/v1/users?page=2",
    "last": "/api/v1/users?page=8"
  }
}

// ═══ 失败 ═══
{
  "code": 42201,
  "message": "参数验证失败",
  "errors": [
    { "field": "name", "message": "姓名不能为空" },
    { "field": "email", "message": "邮箱格式不正确" }
  ]
}
```

---

## 6. 错误码体系

```php
/**
 * 错误码设计：4位业务码 + 2位子码
 * 
 * 例：40101 = 401(未认证) + 01(token过期)
 *     42201 = 422(参数) + 01(邮箱格式错误)
 */
final class ErrorCode
{
    // 通用
    const SUCCESS      = 0;
    const UNKNOWN      = 10000;

    // 认证 401xx
    const UNAUTHORIZED    = 40100;
    const TOKEN_EXPIRED   = 40101;
    const TOKEN_INVALID   = 40102;

    // 授权 403xx
    const FORBIDDEN       = 40300;
    const NO_PERMISSION   = 40301;

    // 资源 404xx
    const NOT_FOUND       = 40400;
    const USER_NOT_FOUND  = 40401;

    // 验证 422xx
    const VALIDATION_FAILED  = 42200;
    const FIELD_REQUIRED     = 42201;
    const FIELD_EMAIL        = 42202;
    const FIELD_MIN_LENGTH   = 42203;
    const FIELD_MAX_LENGTH   = 42204;
    const FIELD_UNIQUE       = 42205;

    // 业务 4xxxx
    const ORDER_PAID         = 40001;  // 订单已支付
    const STOCK_INSUFFICIENT = 40002;  // 库存不足
    const SMS_TOO_FREQUENT   = 40003;  // 短信发送太频繁

    // 限流 429xx
    const TOO_MANY_REQUESTS  = 42900;

    // 服务端 500xx
    const SERVER_ERROR       = 50000;

    public static string $messages = [
        self::TOKEN_EXPIRED => 'token 已过期',
        self::STOCK_INSUFFICIENT => '库存不足',
        // ...
    ];
}
```

---

## 7. 分页设计

### 7.1 偏移分页（最常见）

```php
GET /api/v1/users?page=2&per_page=20

{
  "meta": {
    "current_page": 2,
    "per_page": 20,
    "total": 1000,
    "total_pages": 50
  }
}
```

### 7.2 游标分页（数据量很大时）

```php
// 对大数据集用游标分页，避免 OFFSET 性能问题
GET /api/v1/users?cursor=eyJpZCI6MTAwfQ&per_page=20

// 原理：用上一页最后一条数据的 id/时间 作为游标
// SELECT * FROM users WHERE id > 100 ORDER BY id LIMIT 20
```

```php
// ═══ PHP 游标分页实现 ═══
final class CursorPaginator
{
    public function paginate(PDO $pdo, string $table, ?string $cursor, int $perPage): array
    {
        $where = '';
        $params = [];

        if ($cursor) {
            $decoded = json_decode(base64_decode($cursor), true);
            $where = 'WHERE id > :cursor_id';
            $params['cursor_id'] = $decoded['id'];
        }

        $stmt = $pdo->prepare("SELECT * FROM {$table} {$where} ORDER BY id ASC LIMIT :limit + 1");
        $stmt->bindValue('limit', $perPage + 1, PDO::PARAM_INT);
        foreach ($params as $k => $v) $stmt->bindValue($k, $v);
        $stmt->execute();

        $rows = $stmt->fetchAll(PDO::FETCH_ASSOC);
        $hasMore = count($rows) > $perPage;
        if ($hasMore) array_pop($rows);

        $nextCursor = null;
        if ($hasMore) {
            $last = end($rows);
            $nextCursor = base64_encode(json_encode(['id' => $last['id']]));
        }

        return [
            'data'       => $rows,
            'next_cursor' => $nextCursor,
            'has_more'   => $hasMore,
        ];
    }
}
```

---

## 8. 版本管理

```php
// 方案1：URL 路径（推荐）
GET /api/v1/users
GET /api/v2/users

// 方案2：Header
GET /api/users
Header: Accept: application/vnd.example.v2+json

// 方案3：Query 参数
GET /api/users?version=v2

// ═══ 路由实现 ═══
// 目录结构
// routes/api/v1/user.php
// routes/api/v2/user.php

// Nginx 路由规则
rewrite ^/api/v1/(.*)$ /api/v1/index.php?path=$1 last;
rewrite ^/api/v2/(.*)$ /api/v2/index.php?path=$1 last;
```

---

## 9. 认证与授权

```php
// ═══ 标准化认证头 ═══
Authorization: Bearer <access_token>

// ═══ 中间件实现 ═══
final class AuthMiddleware
{
    public function handle(array $request, callable $next): array
    {
        $header = $_SERVER['HTTP_AUTHORIZATION'] ?? '';

        if (!preg_match('/^Bearer (.+)$/', $header, $matches)) {
            return $this->error(40100, '未提供 token');
        }

        $token = $matches[1];
        $payload = (new JwtService())->verify($token);

        if ($payload === null) {
            return $this->error(40101, 'token 无效或过期');
        }

        // 注入当前用户
        $request['auth_user'] = $payload;

        return $next($request);
    }
}
```

---

## 10. 接口文档

```yaml
# OpenAPI 3.0 (Swagger) 示例
openapi: 3.0.0
info:
  title: 电商 API
  version: 1.0.0
paths:
  /api/v1/users/{id}:
    get:
      summary: 获取用户详情
      tags: [用户]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  code:
                    type: integer
                    example: 0
                  data:
                    $ref: '#/components/schemas/User'
        '404':
          description: 用户不存在

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
        name:
          type: string
        email:
          type: string
          format: email
```

---

## 11. 限流设计

```php
// ═══ 令牌桶限流 ═══
final class RateLimiter
{
    private Redis $redis;

    public function __construct()
    {
        $this->redis = new Redis();
        $this->redis->connect('127.0.0.1', 6379);
    }

    public function check(string $key, int $maxRequests, int $windowSeconds): bool
    {
        $now = microtime(true);
        $windowKey = "rate_limit:{$key}:" . floor($now / $windowSeconds);

        $count = $this->redis->incr($windowKey);
        $this->redis->expire($windowKey, $windowSeconds + 1);

        return $count <= $maxRequests;
    }
}

// 中间件使用
$limiter = new RateLimiter();
$userId = $authUser->sub ?? 'anonymous';

if (!$limiter->check("user:{$userId}", 60, 60)) {
    http_response_code(429);
    echo json_encode(['code' => 42900, 'message' => '请求过于频繁']);
    exit;
}

// ═══ 响应头返回剩余次数 ═══
header('X-RateLimit-Limit: 60');
header('X-RateLimit-Remaining: ' . (60 - $count));
header('X-RateLimit-Reset: ' . ($now - ($now % 60) + 60));
```

---

## 12. PHP 完整实现

```php
// ═══ 轻量 API 框架骨架 ═══
final class ApiResponse
{
    public static function success(mixed $data = null, array $meta = [], int $status = 200): never
    {
        http_response_code($status);
        $response = ['code' => 0, 'message' => 'ok'];

        if ($data !== null) {
            $response['data'] = $data;
        }
        if ($meta) {
            $response['meta'] = $meta;
        }

        header('Content-Type: application/json');
        echo json_encode($response, JSON_UNESCAPED_UNICODE);
        exit;
    }

    public static function error(int $code, string $message, array $errors = [], int $status = 400): never
    {
        http_response_code($status);
        $response = ['code' => $code, 'message' => $message];
        if ($errors) {
            $response['errors'] = $errors;
        }

        header('Content-Type: application/json');
        echo json_encode($response, JSON_UNESCAPED_UNICODE);
        exit;
    }
}

// ═══ 路由分发 ═══
final class Router
{
    private array $routes = [];

    public function get(string $path, callable $handler): void
    {
        $this->routes['GET'][$path] = $handler;
    }

    public function post(string $path, callable $handler): void
    {
        $this->routes['POST'][$path] = $handler;
    }

    public function put(string $path, callable $handler): void
    {
        $this->routes['PUT'][$path] = $handler;
    }

    public function delete(string $path, callable $handler): void
    {
        $this->routes['DELETE'][$path] = $handler;
    }

    public function dispatch(): void
    {
        $method = $_SERVER['REQUEST_METHOD'];
        $uri = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);

        // 查找路由（简化版，实际项目用 FastRoute）
        foreach (($this->routes[$method] ?? []) as $route => $handler) {
            $pattern = preg_replace('/\{(\w+)\}/', '(?P<$1>[^/]+)', $route);
            if (preg_match("#^{$pattern}$#", $uri, $matches)) {
                $params = array_filter($matches, 'is_string', ARRAY_FILTER_USE_KEY);
                echo json_encode($handler($params, $this->getRequestData()));
                return;
            }
        }

        ApiResponse::error(40400, '接口不存在', [], 404);
    }

    private function getRequestData(): array
    {
        $body = json_decode(file_get_contents('php://input'), true) ?? [];
        return array_merge($_GET, $body);
    }
}

// ═══ 运行 ═══
$router = new Router();

// 用户列表
$router->get('/api/v1/users', function ($params, $input) {
    $page  = (int)($input['page'] ?? 1);
    $perPage = min((int)($input['per_page'] ?? 20), 100);

    $pdo = new PDO('mysql:host=127.0.0.1;dbname=mydb', 'root', 'pass');
    $total = $pdo->query('SELECT COUNT(*) FROM users')->fetchColumn();

    $offset = ($page - 1) * $perPage;
    $stmt = $pdo->prepare("SELECT id, name, email, created_at FROM users ORDER BY id DESC LIMIT {$perPage} OFFSET {$offset}");
    $stmt->execute();

    return ApiResponse::success($stmt->fetchAll(PDO::FETCH_ASSOC), [
        'current_page' => $page,
        'per_page'     => $perPage,
        'total'        => (int)$total,
        'total_pages'  => (int)ceil($total / $perPage),
    ]);
});

// 用户详情
$router->get('/api/v1/users/{id}', function ($params, $input) {
    $pdo = new PDO('mysql:host=127.0.0.1;dbname=mydb', 'root', 'pass');
    $stmt = $pdo->prepare('SELECT id, name, email, created_at FROM users WHERE id = :id');
    $stmt->execute(['id' => $params['id']]);
    $user = $stmt->fetch(PDO::FETCH_ASSOC);

    if (!$user) {
        return ApiResponse::error(40401, '用户不存在', [], 404);
    }

    return ApiResponse::success($user);
});

// 创建用户
$router->post('/api/v1/users', function ($params, $input) {
    $errors = [];

    if (empty($input['name'])) {
        $errors[] = ['field' => 'name', 'message' => '姓名不能为空'];
    }
    if (empty($input['email']) || !filter_var($input['email'], FILTER_VALIDATE_EMAIL)) {
        $errors[] = ['field' => 'email', 'message' => '邮箱格式不正确'];
    }

    if ($errors) {
        return ApiResponse::error(42200, '参数验证失败', $errors, 422);
    }

    $pdo = new PDO('mysql:host=127.0.0.1;dbname=mydb', 'root', 'pass');
    $stmt = $pdo->prepare('INSERT INTO users (name, email) VALUES (:name, :email)');
    $stmt->execute(['name' => $input['name'], 'email' => $input['email']]);
    $id = $pdo->lastInsertId();

    return ApiResponse::success(['id' => $id, 'name' => $input['name'], 'email' => $input['email']], status: 201);
});

$router->dispatch();
```

---

> 📡 **好的 API 像好的 UI — 用户一眼就知道怎么用。**
