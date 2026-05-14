# PHP 代码开发规范文档

> 版本：v1.0 | 更新日期：2026-05-13

---

## 目录

1. [命名规范](#1-命名规范)
2. [代码风格](#2-代码风格)
3. [目录结构](#3-目录结构)
4. [错误与异常处理](#4-错误与异常处理)
5. [数据库操作](#5-数据库操作)
6. [安全最佳实践](#6-安全最佳实践)
7. [性能优化](#7-性能优化)
8. [日志规范](#8-日志规范)
9. [测试规范](#9-测试规范)
10. [代码审查清单](#10-代码审查清单)

---

## 1. 命名规范

### 1.1 变量

```php
// ✅ 小驼峰命名 (camelCase)
$userName    = 'khz';
$orderList   = [];
$isActivated = true;

// ❌ 禁止
$user_name;    // 蛇形不用于变量
$UserName;     // 大驼峰不用于变量
$u;            // 无意义的单字母（除循环计数器）
```

### 1.2 常量

```php
// ✅ 大写下划线
const MAX_RETRY_TIMES = 3;
define('APP_ENV', 'production');

// ❌ 禁止
const maxRetryTimes = 3;
define('appEnv', 'production');
```

### 1.3 函数 / 方法

```php
// ✅ 小驼峰，动词开头
function getUserById(int $id): ?User { }
function sendNotification(string $message): void { }
function isEmailValid(string $email): bool { }
function hasPermission(string $role): bool { }

// ❌ 禁止
function get_user_by_id() { }  // 蛇形
function user() { }             // 缺少动词
```

### 1.4 类 / 接口 / Trait

```php
// ✅ 大驼峰 (PascalCase)，名词
class UserController { }
class OrderService { }
interface PaymentGatewayInterface { }
trait Loggable { }

// ❌ 禁止
class user_controller { }
class Orders { } // 复数只在集合类使用
```

### 1.5 数据库

```sql
-- ✅ 表名：蛇形复数
users | orders | order_items | user_roles

-- ✅ 字段名：蛇形小写
id | user_id | order_id | created_at | updated_at | is_deleted

-- ✅ 索引名
idx_user_email       -- 普通索引
uk_user_email        -- 唯一索引
fk_orders_user_id    -- 外键
```

---

## 2. 代码风格

### 2.1 基础格式

```php
<?php

declare(strict_types=1);

namespace App\Services\User;

use App\Models\User;
use App\Exceptions\UserNotFoundException;

/**
 * 用户服务类
 *
 * 负责用户相关的业务逻辑处理
 */
final class UserService
{
    /**
     * 根据ID获取用户信息
     *
     * @param int $userId 用户ID
     * @return User
     * @throws UserNotFoundException
     */
    public function getUserById(int $userId): User
    {
        $user = User::find($userId);

        if (!$user) {
            throw new UserNotFoundException("用户 {$userId} 不存在");
        }

        return $user;
    }
}
```

### 2.2 缩进与括号

```php
// ✅ PSR-12 标准：4空格缩进，括号同行
if ($condition) {
    doSomething();
} else {
    doOther();
}

// 括号内不在首尾加空格
function foo($arg1, $arg2) { }

// ❌ 禁止：Tab缩进、括号另起一行 (Allman风格)
if ($condition)
{
    doSomething();
}
```

### 2.3 运算符

```php
// ✅ 运算符前后各一个空格
$result = $a + $b;
$flag   = $x && $y;

// ✅ 长条件换行
if (
    $condition1
    && $condition2
    && $condition3
) {
    doSomething();
}

// ❌
$result=$a+$b;
```

### 2.4 数组

```php
// ✅ 短数组语法
$list = ['php', 'laravel', 'mysql'];

// ✅ 多行对齐
$config = [
    'driver'   => 'mysql',
    'host'     => '127.0.0.1',
    'charset'  => 'utf8mb4',
    'prefix'   => 'app_',
];

// ❌ 禁止
$list = array('php', 'laravel');
```

### 2.5 类型声明

```php
// ✅ 严格类型
declare(strict_types=1);

// ✅ 参数 + 返回值类型
public function createOrder(
    int $userId,
    array $items,
    ?string $couponCode = null,
): Order { }

// ✅ 可空类型
public function findUser(string $email): ?User { }

// ✅ 联合类型 (PHP 8.0+)
public function format(mixed $value): string|int { }
```

---

## 3. 目录结构

### 3.1 项目结构

```
project/
├── app/                    # 核心应用代码
│   ├── Console/           # 命令行脚本
│   ├── Enums/             # 枚举类
│   ├── Events/            # 事件
│   ├── Exceptions/        # 自定义异常
│   ├── Http/
│   │   ├── Controllers/   # 控制器
│   │   ├── Middleware/    # 中间件
│   │   ├── Requests/      # 表单验证
│   │   └── Resources/     # API 资源
│   ├── Jobs/              # 队列任务
│   ├── Listeners/         # 事件监听器
│   ├── Models/            # 数据模型
│   ├── Services/          # 业务逻辑层
│   ├── Repositories/      # 数据仓储层
│   └── Traits/            # Trait
├── config/                # 配置文件
├── database/
│   ├── migrations/        # 数据迁移
│   ├── seeders/           # 数据填充
│   └── factories/         # 模型工厂
├── public/                # 入口文件
├── routes/                # 路由
├── storage/               # 存储 (日志/缓存/上传)
├── tests/                 # 测试
└── vendor/                # 依赖包
```

### 3.2 控制器精简原则

```php
// ❌ 臃肿的控制器
class OrderController
{
    public function place(Request $request)
    {
        // 100+ 行业务逻辑...
    }
}

// ✅ 控制器只负责调度
final class OrderController
{
    public function __construct(
        private PlaceOrderService $placeOrderService,
    ) {}

    public function place(PlaceOrderRequest $request): OrderResource
    {
        $order = $this->placeOrderService->execute(
            userId: auth()->id(),
            items: $request->validated('items'),
        );

        return new OrderResource($order);
    }
}
```

---

## 4. 错误与异常处理

### 4.1 自定义异常

```php
// ✅ 业务异常分层
abstract class BusinessException extends \RuntimeException
{
    abstract public function getErrorCode(): int;
    abstract public function getHttpCode(): int;
}

final class InsufficientStockException extends BusinessException
{
    public function getErrorCode(): int { return 10001; }
    public function getHttpCode(): int { return 422; }
}

final class ForbiddenException extends BusinessException
{
    public function getErrorCode(): int { return 40300; }
    public function getHttpCode(): int { return 403; }
}
```

### 4.2 异常处理中间件

```php
final class ExceptionHandler
{
    public function render(\Throwable $e): JsonResponse
    {
        // 业务异常：返回友好信息
        if ($e instanceof BusinessException) {
            return response()->json([
                'code'    => $e->getErrorCode(),
                'message' => $e->getMessage(),
            ], $e->getHttpCode());
        }

        // 生产环境隐藏系统错误
        if (app()->isProduction()) {
            Log::error('系统异常', ['exception' => $e]);
            return response()->json([
                'code'    => 50000,
                'message' => '服务器繁忙，请稍后重试',
            ], 500);
        }

        throw $e; // 开发环境显示详情
    }
}
```

### 4.3 不要吞掉异常

```php
// ❌ 吞掉异常
try {
    doSomething();
} catch (\Exception $e) {
    // 什么都不做
}

// ✅ 要么处理，要么记录
try {
    $data = $this->fetchFromApi();
} catch (ApiException $e) {
    Log::warning('API调用失败', ['error' => $e->getMessage()]);
    $data = $this->getFallbackData(); // 降级方案
}
```

---

## 5. 数据库操作

### 5.1 使用参数绑定

```php
// ✅ 参数绑定（防SQL注入）
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email');
$stmt->execute(['email' => $email]);

// Laravel 自动绑定
User::where('email', $email)->first();

// ❌ 禁止：直接拼接
$sql = "SELECT * FROM users WHERE email = '{$email}'"; // 注入风险！
```

### 5.2 避免 N+1 查询

```php
// ❌ N+1 问题
$users = User::all();
foreach ($users as $user) {
    echo $user->orders->count(); // 每条执行一次查询
}

// ✅ 预加载
$users = User::with('orders')->get();
// 或只取需要的关联
$users = User::withCount('orders')->get();
```

### 5.3 字段只取所需

```php
// ❌ 取全部字段
User::all();

// ✅ 指定字段
User::select(['id', 'name', 'email'])->get();

// ✅ 大量数据用游标
User::cursor()->each(function ($user) {
    // 逐条处理，内存友好
});
```

### 5.4 事务处理

```php
// ✅ 事务 + 回滚
DB::transaction(function () use ($orderData) {
    $order = Order::create($orderData);

    foreach ($orderData['items'] as $item) {
        $order->items()->create($item);
    }

    // 扣库存
    Stock::whereIn('id', $skuIds)->decrement('quantity');
}, 3); // 死锁重试3次
```

### 5.5 批量插入与更新

```php
// ✅ 批量插入
OrderItem::insert($itemsArray); // 一次SQL，不触发事件

// ✅ 批量更新 (Laravel)
User::where('status', 'inactive')
    ->update(['status' => 'archived']);

// ❌ 循环单条
foreach ($items as $item) {
    OrderItem::create($item); // 极慢
}
```

---

## 6. 安全最佳实践

### 6.1 输入 & 输出

```php
// ✅ 始终使用验证
final class CreateUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name'     => ['required', 'string', 'max:50'],
            'email'    => ['required', 'email', 'unique:users'],
            'password' => ['required', 'min:8', 'confirmed'],
            'age'      => ['integer', 'min:1', 'max:150'],
            'role'     => [Rule::in(['user', 'admin'])],
        ];
    }
}

// ✅ 模板引擎自动转义
{{ $user->name }}        // Blade 自动转义
<?= htmlspecialchars($name, ENT_QUOTES, 'UTF-8') ?>
```

### 6.2 密码 & 敏感信息

```php
// ✅ 密码哈希
$user->password = password_hash($password, PASSWORD_BCRYPT, ['cost' => 12]);

// ✅ 验证
if (!password_verify($inputPassword, $user->password)) {
    throw new AuthException('密码错误');
}

// ❌ 禁止明文 / MD5
$user->password = md5($password); // 不安全！
```

### 6.3 上传文件

```php
final class FileUploadService
{
    private const ALLOWED_MIME = [
        'image/jpeg' => 'jpg',
        'image/png'  => 'png',
        'image/gif'  => 'gif',
        'image/webp' => 'webp',
    ];

    private const MAX_SIZE = 5 * 1024 * 1024; // 5MB

    public function upload(UploadedFile $file): string
    {
        // 验证类型
        $mime = $file->getMimeType();
        if (!array_key_exists($mime, self::ALLOWED_MIME)) {
            throw new InvalidFileException('不支持的文件类型');
        }

        // 验证大小
        if ($file->getSize() > self::MAX_SIZE) {
            throw new InvalidFileException('文件超过大小限制');
        }

        // 随机文件名
        $filename = bin2hex(random_bytes(16)) . '.' . self::ALLOWED_MIME[$mime];

        // 存储在非公开目录
        $path = $file->storeAs('uploads/' . date('Y/m/d'), $filename, 'private');

        return $path;
    }
}
```

### 6.4 CSRF / XSS

```php
// ✅ Laravel 内置防 CSRF
<form method="POST" action="/orders">
    @csrf
    <!-- ... -->
</form>

// ✅ CSP 头
header("Content-Security-Policy: default-src 'self'; script-src 'self'");

// ✅ Cookie 安全
session_set_cookie_params([
    'lifetime' => 3600,
    'path'     => '/',
    'secure'   => true,       // 仅 HTTPS
    'httponly' => true,       // 禁止 JS 访问
    'samesite' => 'Lax',
]);
```

### 6.5 CORS

```php
// ✅ 严格限制
header('Access-Control-Allow-Origin: https://your-frontend.com');
header('Access-Control-Allow-Methods: GET, POST');
header('Access-Control-Allow-Headers: Content-Type, Authorization');

// ❌ 禁止
header('Access-Control-Allow-Origin: *'); // 不安全！
```

---

## 7. 性能优化

### 7.1 缓存策略

```php
// ✅ 数据缓存
$users = Cache::remember('active_users', 3600, function () {
    return User::where('status', 'active')->get();
});

// ✅ 查询缓存
$config = Cache::rememberForever('system_config', fn() => Config::all());

// ✅ 清除缓存
Cache::tags(['users'])->flush(); // 标签清除
Cache::forget('active_users');    // 单key清除
```

### 7.2 延迟加载 & 惰性计算

```php
// ✅ 惰性集合（减少内存）
$lines = LazyCollection::make(function () {
    $handle = fopen('large.csv', 'r');
    while ($row = fgetcsv($handle)) {
        yield $row;
    }
});

$lines->filter(fn($row) => $row[0] > 100)
      ->each(fn($row) => process($row));
```

### 7.3 队列异步处理

```php
// ✅ 耗时操作放入队列
final class SendWelcomeEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(private User $user) {}

    public function handle(): void
    {
        Mail::to($this->user->email)
            ->send(new WelcomeMail($this->user));
    }
}

// 分发
dispatch(new SendWelcomeEmail($user))->delay(now()->addSeconds(10));
```

### 7.4 数据库索引

```sql
-- ✅ 常用查询加索引
ALTER TABLE orders ADD INDEX idx_user_status (user_id, status);
ALTER TABLE orders ADD INDEX idx_created_at (created_at);

-- ✅ 覆盖索引减少回表
ALTER TABLE users ADD INDEX idx_email_name (email, name);

-- ❌ 避免：在大表直接加索引（会锁表）
-- 使用 pt-online-schema-change 或 gh-ost
```

---

## 8. 日志规范

### 8.1 日志级别

```php
// ✅ 按场景选择级别
Log::emergency('系统不可用，立即处理');     // 最高
Log::alert('支付通道全部断开');
Log::critical('磁盘空间不足');
Log::error('订单创建失败', ['order' => $data]);
Log::warning('API响应超过2秒', ['endpoint' => $url, 'duration' => $ms]);
Log::info('用户登录成功', ['user_id' => $id]);
Log::debug('SQL调试', ['sql' => $sql, 'bindings' => $bindings]);
```

### 8.2 日志格式

```php
// ✅ 结构化日志
Log::info('订单状态变更', [
    'order_id'   => $order->id,
    'from'       => 'pending',
    'to'         => 'paid',
    'amount'     => $order->amount,
    'user_id'    => auth()->id(),
    'ip'         => request()->ip(),
    'trace_id'   => request()->header('X-Trace-Id'),
]);

// ❌ 拼接式日志
Log::info("订单{$orderId}已支付，金额{$amount}");
```

### 8.3 敏感信息脱敏

```php
// ✅ 脱敏后记录
Log::info('用户操作', [
    'user_id'  => $user->id,
    'phone'    => substr($phone, 0, 3) . '****' . substr($phone, -4),
    'action'   => 'update_profile',
]);

// ❌ 禁止日志中输出
// - 密码
// - 完整手机号/身份证
// - Token / Session ID
// - 银行卡号
```

---

## 9. 测试规范

### 9.1 单元测试

```php
final class OrderServiceTest extends TestCase
{
    private OrderService $service;

    protected function setUp(): void
    {
        parent::setUp();
        $this->service = new OrderService();
    }

    /** @test */
    public function it_creates_order_with_valid_data(): void
    {
        // Arrange
        $data = ['user_id' => 1, 'amount' => 99.00];

        // Act
        $order = $this->service->create($data);

        // Assert
        $this->assertInstanceOf(Order::class, $order);
        $this->assertEquals(OrderStatus::PENDING, $order->status);
    }

    /** @test */
    public function it_throws_exception_when_stock_insufficient(): void
    {
        $this->expectException(InsufficientStockException::class);

        $this->service->create($this->oversellData());
    }
}
```

### 9.2 命名约定

```php
// ✅ 清晰描述测试意图
public function it_prevents_duplicate_email_registration(): void { }
public function it_returns_401_when_token_expired(): void { }
public function it_sends_notification_after_order_paid(): void { }

// ❌ 含糊的命名
public function test1(): void { }
public function testUser(): void { }
```

### 9.3 覆盖率目标

| 类型 | 最低覆盖率 | 推荐覆盖率 |
|------|-----------|-----------|
| 核心业务 Service | 90% | 100% |
| Controller | 70% | 85% |
| Model | 60% | 80% |
| 整体 | 75% | 85% |

---

## 10. 代码审查清单

### 提交前自查

- [ ] 代码通过所有单元测试
- [ ] 没有遗留的 `dd()` / `var_dump()` / `print_r()`
- [ ] 没有硬编码的密钥/密码
- [ ] 新功能有对应的测试用例
- [ ] 数据库变更包含 migration + rollback
- [ ] 对外 API 有版本兼容处理
- [ ] 错误处理完整，无吞异常
- [ ] 日志输出合理，敏感信息已脱敏
- [ ] 命名清晰，见名知义
- [ ] 无未使用的 import / 变量
- [ ] 符合 PSR-12 代码风格

### Review 关注点

1. **安全性**：是否有注入、越权、数据泄漏风险
2. **性能**：是否产生 N+1 查询、是否缺缓存、是否需要队列
3. **可维护性**：逻辑是否清晰、是否有足够注释、是否利于扩展
4. **边界情况**：空值、极端值、并发情况是否处理

---

## 附录

### A. 推荐工具

| 工具 | 用途 |
|------|------|
| PHPStan / Psalm | 静态分析 |
| PHP CS Fixer | 代码格式化 |
| PHPUnit | 单元测试 |
| Laravel Telescope | 调试监控 |
| Blackfire | 性能分析 |
| Rector | 自动升级/重构 |

### B. PHP 版本特性

| 版本 | 推荐使用的特性 |
|------|---------------|
| 8.0 | 命名参数、联合类型、match 表达式、nullsafe、属性注解 |
| 8.1 | 枚举、readonly 属性、纤程 Fiber、array_is_list |
| 8.2 | readonly 类、独立类型 null/false/true、敏感参数 |
| 8.3 | 类型化常量、json_validate()、override 注解 |
| 8.4 | 属性钩子、非对称可见性、新数组函数 |

### C. 参考链接

- [PSR-1: 基础编码规范](https://www.php-fig.org/psr/psr-1/)
- [PSR-4: 自动加载规范](https://www.php-fig.org/psr/psr-4/)
- [PSR-12: 扩展编码风格](https://www.php-fig.org/psr/psr-12/)
- [OWASP PHP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/PHP_Configuration_Cheat_Sheet.html)
- [PHP The Right Way](https://phptherightway.com/)

---

> 📝 **编写代码是为了让人读懂，其次才是让机器执行。**
