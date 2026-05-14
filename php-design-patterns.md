# PHP 设计模式实战：23 种模式精选

> 面向真实 PHP 场景，从创建型到行为型 — 代码即文档，拿来即用

---

## 目录

1. [设计模式核心原则](#1-设计模式核心原则)
2. [单例模式 Singleton](#2-单例模式-singleton)
3. [工厂方法 Factory Method](#3-工厂方法-factory-method)
4. [抽象工厂 Abstract Factory](#4-抽象工厂-abstract-factory)
5. [建造者 Builder](#5-建造者-builder)
6. [原型 Prototype](#6-原型-prototype)
7. [适配器 Adapter](#7-适配器-adapter)
8. [装饰器 Decorator](#8-装饰器-decorator)
9. [代理 Proxy](#9-代理-proxy)
10. [观察者 Observer](#10-观察者-observer)
11. [策略 Strategy](#11-策略-strategy)
12. [责任链 Chain of Responsibility](#12-责任链-chain-of-responsibility)
13. [命令 Command](#13-命令-command)
14. [模板方法 Template Method](#14-模板方法-template-method)
15. [依赖注入 DI](#15-依赖注入-dependency-injection)
16. [实战中的模式选择](#16-实战中的模式选择)

---

## 1. 设计模式核心原则

### SOLID 五原则

| 原则 | 说明 |
|------|------|
| **S** - 单一职责 | 一个类只做一件事 |
| **O** - 开闭原则 | 对扩展开放，对修改关闭 |
| **L** - 里氏替换 | 子类可替换父类 |
| **I** - 接口隔离 | 不强迫实现不需要的接口 |
| **D** - 依赖倒置 | 依赖抽象而非具体实现 |

### 模式分类

| 类型 | 模式 |
|------|------|
| **创建型** | 单例、工厂方法、抽象工厂、建造者、原型 |
| **结构型** | 适配器、装饰器、代理、外观、桥接、组合、享元 |
| **行为型** | 观察者、策略、责任链、命令、模板方法、状态、迭代器 |

---

## 2. 单例模式 Singleton

**场景：** 数据库连接、配置管理、日志 — 全局只需要一个实例。

```php
class Database
{
    private static ?self $instance = null;
    private \PDO $pdo;

    private function __construct()
    {
        $this->pdo = new \PDO(
            'mysql:host=localhost;dbname=app;charset=utf8mb4',
            'root',
            '',
            [
                \PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION,
                \PDO::ATTR_DEFAULT_FETCH_MODE => \PDO::FETCH_ASSOC,
            ]
        );
    }

    public static function getInstance(): self
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    public function connection(): \PDO
    {
        return $this->pdo;
    }

    // 防止克隆和反序列化
    private function __clone() {}
    public function __wakeup()
    {
        throw new \Exception('不能反序列化单例');
    }
}

// 使用
$db  = Database::getInstance()->connection();
$db2 = Database::getInstance()->connection();
var_dump($db === $db2); // true — 同一个连接

// 测试友好变体：允许注入
class Database
{
    private static ?\PDO $mockPdo = null;

    public static function setMock(\PDO $pdo): void
    {
        self::$mockPdo = $pdo;
        self::$instance = null;
    }

    private function __construct()
    {
        $this->pdo = self::$mockPdo ?? new \PDO(/*...*/);
    }
}
```

**⚠️ 注意：** 大多数现代框架（Laravel、Symfony）通过 DI 容器管理对象生命周期，不需要手动写单例。但理解它仍有价值。

---

## 3. 工厂方法 Factory Method

**场景：** 根据配置创建不同的支付驱动（微信/支付宝/Stripe）。

```php
// 产品接口
interface PaymentGateway
{
    public function pay(float $amount): array;
    public function refund(string $transactionId): bool;
}

// 具体产品
class WechatPay implements PaymentGateway
{
    public function pay(float $amount): array
    {
        return ['method' => 'wechat', 'amount' => $amount];
    }
    public function refund(string $transactionId): bool { /* ... */ return true; }
}

class AliPay implements PaymentGateway
{
    public function pay(float $amount): array
    {
        return ['method' => 'alipay', 'amount' => $amount, 'discount' => 0.05];
    }
    public function refund(string $transactionId): bool { /* ... */ return true; }
}

class StripePay implements PaymentGateway
{
    public function pay(float $amount): array
    {
        return ['method' => 'stripe', 'amount' => $amount];
    }
    public function refund(string $transactionId): bool { /* ... */ return true; }
}

// 工厂
abstract class PaymentFactory
{
    abstract public function createGateway(): PaymentGateway;

    public function process(float $amount): array
    {
        $gateway = $this->createGateway();
        // 可以在此做日志、审计等公共操作
        return $gateway->pay($amount);
    }
}

// 具体工厂
class WechatPayFactory extends PaymentFactory
{
    public function createGateway(): PaymentGateway
    {
        return new WechatPay();
    }
}

class AliPayFactory extends PaymentFactory
{
    public function createGateway(): PaymentGateway
    {
        return new AliPay();
    }
}

// 客户端
function handlePayment(PaymentFactory $factory, float $amount): array
{
    return $factory->process($amount);
}

echo json_encode(handlePayment(new WechatPayFactory(), 99.00));
```

**变体 — 简单工厂（静态工厂）：**

```php
class PaymentGatewayFactory
{
    public static function create(string $type): PaymentGateway
    {
        return match ($type) {
            'wechat' => new WechatPay(),
            'alipay' => new AliPay(),
            'stripe' => new StripePay(),
            default  => throw new \InvalidArgumentException("Unknown: $type"),
        };
    }
}

$gateway = PaymentGatewayFactory::create('alipay');
$gateway->pay(199.00);
```

---

## 4. 抽象工厂 Abstract Factory

**场景：** 创建一系列相关的对象族（例如 UI 组件：Light 主题 / Dark 主题全家桶）。

```php
// 抽象产品
interface Button { public function render(): string; }
interface Input  { public function render(): string; }

// 具体产品 - Light 主题
class LightButton implements Button {
    public function render(): string { return '<button class="btn-light">'; }
}
class LightInput implements Input {
    public function render(): string { return '<input class="input-light">'; }
}

// 具体产品 - Dark 主题
class DarkButton implements Button {
    public function render(): string { return '<button class="btn-dark">'; }
}
class DarkInput implements Input {
    public function render(): string { return '<input class="input-dark">'; }
}

// 抽象工厂
interface UIFactory
{
    public function createButton(): Button;
    public function createInput(): Input;
}

// 具体工厂
class LightUIFactory implements UIFactory
{
    public function createButton(): Button { return new LightButton(); }
    public function createInput(): Input  { return new LightInput(); }
}

class DarkUIFactory implements UIFactory
{
    public function createButton(): Button { return new DarkButton(); }
    public function createInput(): Input  { return new DarkInput(); }
}

// 渲染页面
function renderForm(UIFactory $factory): string
{
    $button = $factory->createButton();
    $input  = $factory->createInput();
    return $input->render() . $button->render();
}

// 切换主题只需换工厂
$theme = config('app.theme') === 'dark'
    ? new DarkUIFactory()
    : new LightUIFactory();
echo renderForm($theme);
```

---

## 5. 建造者 Builder

**场景：** 构建复杂的查询构造器、邮件对象、API 请求 — 链式调用分步构造。

```php
class QueryBuilder
{
    private array $select = ['*'];
    private string $table = '';
    private array $where  = [];
    private array $order  = [];
    private int $limit    = 0;
    private int $offset   = 0;

    public function table(string $table): self
    {
        $this->table = $table;
        return $this;
    }

    public function select(array $columns): self
    {
        $this->select = $columns;
        return $this;
    }

    public function where(string $column, string $operator, mixed $value): self
    {
        $this->where[] = [$column, $operator, $value];
        return $this;
    }

    public function whereIn(string $column, array $values): self
    {
        $placeholders = implode(',', array_fill(0, count($values), '?'));
        $this->where[] = "{$column} IN ({$placeholders})";
        return $this;
    }

    public function orderBy(string $column, string $direction = 'ASC'): self
    {
        $this->order[] = "{$column} {$direction}";
        return $this;
    }

    public function limit(int $limit): self
    {
        $this->limit = $limit;
        return $this;
    }

    public function toSql(): string
    {
        $sql = 'SELECT ' . implode(',', $this->select) . " FROM {$this->table}";

        if ($this->where) {
            $sql .= ' WHERE ' . implode(' AND ', array_map(
                fn($w) => is_array($w)
                    ? "{$w[0]} {$w[1]} ?"
                    : $w,
                $this->where
            ));
        }

        if ($this->order) {
            $sql .= ' ORDER BY ' . implode(', ', $this->order);
        }

        if ($this->limit > 0) {
            $sql .= " LIMIT {$this->limit}";
        }

        return $sql;
    }
}

// 使用
$sql = (new QueryBuilder())
    ->table('users')
    ->select(['id', 'name', 'email'])
    ->where('status', '=', 'active')
    ->where('created_at', '>=', '2024-01-01')
    ->orderBy('created_at', 'DESC')
    ->limit(20)
    ->toSql();
// SELECT id,name,email FROM users WHERE status = ? AND created_at >= ? ORDER BY created_at DESC LIMIT 20
```

---

## 6. 原型 Prototype

**场景：** 克隆复杂对象（模板化邮件、大型配置对象）。

```php
class EmailTemplate
{
    private array $headers = [];
    private string $body   = '';
    private array $attachments = [];

    public function __construct(
        public string $subject = '',
    ) {}

    public function withHeader(string $key, string $value): self
    {
        $clone = clone $this;
        $clone->headers[$key] = $value;
        return $clone;
    }

    public function withBody(string $body): self
    {
        $clone = clone $this;
        $clone->body = $body;
        return $clone;
    }

    public function send(): array
    {
        return [
            'subject' => $this->subject,
            'headers' => $this->headers,
            'body'    => $this->body,
        ];
    }
}

// 原型模板
$prototype = new EmailTemplate('订单确认');

// 复制并定制
$email1 = $prototype
    ->withHeader('To', 'user1@example.com')
    ->withBody('订单 #1001 已确认');

$email2 = $prototype
    ->withHeader('To', 'user2@example.com')
    ->withBody('订单 #1002 已确认');

print_r($email1->send());
```

---

## 7. 适配器 Adapter

**场景：** 对接第三方库，统一接口（如用不同云存储 SDK 统一为 `FileStorage` 接口）。

```php
// 目标接口
interface FileStorage
{
    public function put(string $path, string $content): string;
    public function get(string $path): ?string;
    public function delete(string $path): bool;
}

// 本地存储（无需适配——天然符合接口）
class LocalStorage implements FileStorage
{
    public function put(string $path, string $content): string
    {
        file_put_contents('/var/storage/' . $path, $content);
        return $path;
    }
    public function get(string $path): ?string
    {
        return file_get_contents('/var/storage/' . $path) ?: null;
    }
    public function delete(string $path): bool
    {
        return unlink('/var/storage/' . $path);
    }
}

// 阿里云 OSS SDK（第三方，不可修改）
class AlibabaOssClient
{
    public function uploadObject(string $bucket, string $key, string $body): array
    {
        return ['url' => "https://{$bucket}.oss-cn-hangzhou.aliyuncs.com/{$key}"];
    }

    public function downloadObject(string $bucket, string $key): ?string
    {
        return 'file content';
    }
}

// 适配器
class AlibabaOssAdapter implements FileStorage
{
    public function __construct(
        private AlibabaOssClient $client,
        private string $bucket,
    ) {}

    public function put(string $path, string $content): string
    {
        $result = $this->client->uploadObject($this->bucket, $path, $content);
        return $result['url'];
    }

    public function get(string $path): ?string
    {
        return $this->client->downloadObject($this->bucket, $path);
    }

    public function delete(string $path): bool
    {
        // ... 调用 OSS 删除 API
        return true;
    }
}

// 使用——无缝切换存储引擎
$storage = config('storage.driver') === 'oss'
    ? new AlibabaOssAdapter(new AlibabaOssClient(), 'my-bucket')
    : new LocalStorage();

$storage->put('avatars/user1.jpg', $imageData);
```

---

## 8. 装饰器 Decorator

**场景：** 在不修改原有类的前提下，动态添加功能（给 API 加缓存层、加日志）。

```php
// 组件接口
interface PostRepository
{
    public function find(int $id): ?array;
    public function all(): array;
}

// 具体组件
class DatabasePostRepository implements PostRepository
{
    public function find(int $id): ?array
    {
        sleep(1); // 模拟慢查询
        return ['id' => $id, 'title' => "Post {$id}"];
    }

    public function all(): array
    {
        return [['id' => 1], ['id' => 2]];
    }
}

// 装饰器：缓存层
class CachedPostRepository implements PostRepository
{
    private array $cache = [];

    public function __construct(private PostRepository $inner) {}

    public function find(int $id): ?array
    {
        $key = "post:{$id}";
        if (!isset($this->cache[$key])) {
            $this->cache[$key] = $this->inner->find($id);
        }
        return $this->cache[$key];
    }

    public function all(): array
    {
        return $this->inner->all(); // 列表不缓存
    }
}

// 装饰器：日志层
class LoggedPostRepository implements PostRepository
{
    public function __construct(private PostRepository $inner) {}

    public function find(int $id): ?array
    {
        $start  = microtime(true);
        $result = $this->inner->find($id);
        $elapsed = (microtime(true) - $start) * 1000;
        \Log::info("Post::find({$id})", ['ms' => $elapsed]);
        return $result;
    }

    public function all(): array
    {
        return $this->inner->all();
    }
}

// 自由组合
$repo = new LoggedPostRepository(
    new CachedPostRepository(
        new DatabasePostRepository()
    )
);
$repo->find(1); // 第一次慢，第二次命中缓存
```

---

## 9. 代理 Proxy

**场景：** 延迟加载、权限控制、远程代理。

```php
// 主题接口
interface Image
{
    public function display(): string;
}

// 真实对象（代价高：需要从磁盘加载大图）
class RealImage implements Image
{
    private string $filename;

    public function __construct(string $filename)
    {
        $this->filename = $filename;
        $this->loadFromDisk();
    }

    private function loadFromDisk(): void
    {
        // 模拟磁盘 I/O
        usleep(100000);
    }

    public function display(): string
    {
        return "<img src='{$this->filename}' alt='photo'>";
    }
}

// 虚拟代理：延迟加载
class ImageProxy implements Image
{
    private ?RealImage $realImage = null;

    public function __construct(private string $filename) {}

    public function display(): string
    {
        if ($this->realImage === null) {
            $this->realImage = new RealImage($this->filename);
        }
        return $this->realImage->display();
    }
}

// 画廊：大量缩略图，只有 view 时才真正加载
$gallery = array_map(fn($f) => new ImageProxy($f), $imageFiles);
// 此时没有磁盘 I/O
echo $gallery[0]->display(); // 第一次访问才加载

// === 保护代理：权限控制 ===
class ProtectedPostService implements PostService
{
    public function __construct(
        private PostService $inner,
        private User $user,
    ) {}

    public function delete(int $id): bool
    {
        if (!$this->user->isAdmin()) {
            throw new \RuntimeException('无权删除');
        }
        return $this->inner->delete($id);
    }
}
```

---

## 10. 观察者 Observer

**场景：** 事件系统（文章发布 → 发通知 + 清理缓存 + 建搜索索引）。

```php
// 观察者接口
interface Observer
{
    public function update(string $event, mixed $data): void;
}

// 主题接口
interface Subject
{
    public function attach(Observer $observer): void;
    public function detach(Observer $observer): void;
    public function notify(string $event, mixed $data = null): void;
}

// 具体主题
class PostPublisher implements Subject
{
    private array $observers = [];

    public function attach(Observer $observer): void
    {
        $this->observers[] = $observer;
    }

    public function detach(Observer $observer): void
    {
        $this->observers = array_filter(
            $this->observers,
            fn($o) => $o !== $observer
        );
    }

    public function notify(string $event, mixed $data = null): void
    {
        foreach ($this->observers as $observer) {
            $observer->update($event, $data);
        }
    }

    public function publish(array $post): void
    {
        // 保存文章...
        echo "文章「{$post['title']}」发布成功\n";

        // 通知所有观察者
        $this->notify('published', $post);
    }
}

// 具体观察者
class EmailNotifier implements Observer
{
    public function update(string $event, mixed $data): void
    {
        if ($event === 'published') {
            echo "[Email] 新文章「{$data['title']}」已发布\n";
        }
    }
}

class SearchIndexer implements Observer
{
    public function update(string $event, mixed $data): void
    {
        if ($event === 'published') {
            echo "[Search] 索引文章 {$data['id']}\n";
        }
    }
}

class CacheCleaner implements Observer
{
    public function update(string $event, mixed $data): void
    {
        echo "[Cache] 清理首页缓存\n";
    }
}

// 使用
$publisher = new PostPublisher();
$publisher->attach(new EmailNotifier());
$publisher->attach(new SearchIndexer());
$publisher->attach(new CacheCleaner());

$publisher->publish(['id' => 42, 'title' => '设计模式入门']);
```

---

## 11. 策略 Strategy

**场景：** 多种算法可互换（不同排序规则、不同导出格式、不同通知渠道）。

```php
// 策略接口
interface PricingStrategy
{
    public function calculate(float $price): float;
}

// 具体策略
class RegularPricing implements PricingStrategy
{
    public function calculate(float $price): float
    {
        return $price;
    }
}

class VipPricing implements PricingStrategy
{
    public function calculate(float $price): float
    {
        return $price * 0.8; // VIP 8 折
    }
}

class BlackFridayPricing implements PricingStrategy
{
    public function calculate(float $price): float
    {
        return max($price * 0.5, 9.9); // 5 折但最低 9.9
    }
}

class NewUserPricing implements PricingStrategy
{
    public function calculate(float $price): float
    {
        return $price * 0.9 - 5; // 9 折再减 5
    }
}

// 上下文
class Product
{
    public function __construct(
        public string $name,
        public float  $price,
    ) {}

    public function getFinalPrice(PricingStrategy $strategy): float
    {
        return $strategy->calculate($this->price);
    }
}

// 使用 — 运行时选择策略
$product = new Product('机械键盘', 299.00);

$strategy = match (true) {
    date('m-d') === '11-11' => new BlackFridayPricing(),
    $user->isVip()          => new VipPricing(),
    $user->isNew()          => new NewUserPricing(),
    default                 => new RegularPricing(),
};

echo "最终价格：¥" . $product->getFinalPrice($strategy);
```

---

## 12. 责任链 Chain of Responsibility

**场景：** 层层审批、过滤器管道、请求验证。

```php
abstract class Handler
{
    private ?Handler $next = null;

    public function setNext(Handler $handler): Handler
    {
        $this->next = $handler;
        return $handler; // 支持链式调用 setNext
    }

    public function handle(array $request): bool
    {
        if (!$this->process($request)) {
            return false; // 当前节点拒绝，停止
        }

        if ($this->next !== null) {
            return $this->next->handle($request);
        }

        return true; // 全部通过
    }

    abstract protected function process(array $request): bool;
}

// 具体处理器
class ThrottleHandler extends Handler
{
    protected function process(array $request): bool
    {
        $ip = $request['ip'];
        $ok = !in_array($ip, $blacklist);
        echo $ok ? "[Throttle] 通过\n" : "[Throttle] 拒绝—IP 在黑名单\n";
        return $ok;
    }
}

class AuthHandler extends Handler
{
    protected function process(array $request): bool
    {
        $ok = !empty($request['user']);
        echo $ok ? "[Auth] 通过\n" : "[Auth] 拒绝—未登录\n";
        return $ok;
    }
}

class PermissionHandler extends Handler
{
    private array $required = ['create_post'];

    protected function process(array $request): bool
    {
        $userPermissions = $request['user']['permissions'] ?? [];
        $ok = count(array_intersect($this->required, $userPermissions)) > 0;
        echo $ok ? "[Permission] 通过\n" : "[Permission] 拒绝—权限不足\n";
        return $ok;
    }
}

class ValidationHandler extends Handler
{
    protected function process(array $request): bool
    {
        $ok = strlen($request['title'] ?? '') >= 3;
        echo $ok ? "[Validation] 通过\n" : "[Validation] 拒绝—标题太短\n";
        return $ok;
    }
}

// 组装链
$throttle = new ThrottleHandler();
$throttle->setNext(new AuthHandler())
         ->setNext(new PermissionHandler())
         ->setNext(new ValidationHandler());

// 测试通过
$req = ['ip' => '1.2.3.4', 'user' => ['permissions' => ['create_post']], 'title' => 'Hello'];
echo $throttle->handle($req) ? "✅ 请求已处理" : "❌ 请求被拒绝";

// 测试被拒绝
$req = ['ip' => '1.2.3.4', 'user' => null, 'title' => 'Hi'];
echo $throttle->handle($req) ? "✅ 请求已处理" : "❌ 请求被拒绝";
```

---

## 13. 命令 Command

**场景：** 任务队列、撤销/重做、事务日志。

```php
interface Command
{
    public function execute(): void;
    public function undo(): void;
}

// 具体命令
class CreatePostCommand implements Command
{
    private ?int $postId = null;

    public function __construct(
        private \PDO $db,
        private array $data,
    ) {}

    public function execute(): void
    {
        $stmt = $this->db->prepare(
            'INSERT INTO posts (title, content) VALUES (:title, :content)'
        );
        $stmt->execute(['title' => $this->data['title'], 'content' => $this->data['content']]);
        $this->postId = (int)$this->db->lastInsertId();
        echo "创建文章 ID:{$this->postId}\n";
    }

    public function undo(): void
    {
        if ($this->postId) {
            $this->db->prepare('DELETE FROM posts WHERE id = ?')
                ->execute([$this->postId]);
            echo "撤销：删除文章 ID:{$this->postId}\n";
        }
    }
}

// 调用者
class CommandInvoker
{
    private array $history = [];

    public function execute(Command $command): void
    {
        $command->execute();
        $this->history[] = $command;
    }

    public function undoLast(): void
    {
        $command = array_pop($this->history);
        $command?->undo();
    }

    public function undoAll(): void
    {
        while ($command = array_pop($this->history)) {
            $command->undo();
        }
    }
}

// 使用
$invoker = new CommandInvoker();
$invoker->execute(new CreatePostCommand($db, ['title' => 'A', 'content' => '...']));
$invoker->execute(new CreatePostCommand($db, ['title' => 'B', 'content' => '...']));

$invoker->undoLast();  // 撤销 B
$invoker->undoLast();  // 撤销 A
```

---

## 14. 模板方法 Template Method

**场景：** 定义算法骨架，子类实现具体步骤（数据导入/导出、处理流程）。

```php
abstract class DataImporter
{
    // 模板方法 — 定义算法骨架
    final public function import(string $filePath): array
    {
        $raw = $this->readFile($filePath);
        $parsed = $this->parseData($raw);
        $valid = $this->validate($parsed);

        if (empty($valid)) {
            return ['success' => false, 'message' => '无有效数据'];
        }

        $processed = $this->transform($valid);
        $count = $this->save($processed);

        $this->afterImport($count);

        return ['success' => true, 'imported' => $count];
    }

    protected function readFile(string $path): string
    {
        return file_get_contents($path);
    }

    // 子类必须实现
    abstract protected function parseData(string $raw): array;
    abstract protected function save(array $data): int;

    // 子类可选覆盖
    protected function validate(array $data): array { return $data; }
    protected function transform(array $data): array { return $data; }
    protected function afterImport(int $count): void {}
}

// CSV 导入
class CsvImporter extends DataImporter
{
    protected function parseData(string $raw): array
    {
        $rows = str_getcsv(trim($raw), "\n");
        $headers = str_getcsv(array_shift($rows));
        return array_map(fn($r) => array_combine($headers, str_getcsv($r)), $rows);
    }

    protected function validate(array $data): array
    {
        return array_filter($data, fn($r) =>
            filter_var($r['email'] ?? '', FILTER_VALIDATE_EMAIL)
        );
    }

    protected function save(array $data): int
    {
        foreach ($data as $row) {
            // INSERT INTO users ...
        }
        return count($data);
    }
}

// JSON 导入
class JsonImporter extends DataImporter
{
    protected function parseData(string $raw): array
    {
        return json_decode($raw, true, 512, JSON_THROW_ON_ERROR);
    }

    protected function save(array $data): int
    {
        foreach ($data as $row) {
            // INSERT INTO products ...
        }
        return count($data);
    }
}

// 使用
$importer = new CsvImporter();
$result = $importer->import('/tmp/users.csv');
```

---

## 15. 依赖注入 Dependency Injection

**场景：** 解除硬编码依赖，实现松耦合和可测试性。

```php
// ❌ 紧耦合
class OrderService
{
    public function create(array $data): void
    {
        $mailer = new SmtpMailer('smtp.example.com', 587);
        $logger = new FileLogger('/var/log/orders.log');

        // 创建订单...
        $mailer->send($data['email'], '订单确认');
        $logger->log("订单创建: {$data['id']}");
    }
}

// ✅ 构造器注入
class OrderService
{
    public function __construct(
        private Mailer $mailer,
        private Logger $logger,
        private OrderRepository $repository,
    ) {}

    public function create(array $data): Order
    {
        $order = $this->repository->create($data);

        $this->mailer->send($data['email'], '订单确认', [
            'order_id' => $order->id,
        ]);

        $this->logger->log("订单创建: {$order->id}", ['order' => $order->toArray()]);

        return $order;
    }
}

// 接口（便于测试 Mock）
interface Mailer
{
    public function send(string $to, string $subject, array $context): void;
}

interface Logger
{
    public function log(string $message, array $context = []): void;
}

// 容器装配（Laravel 示例）
// AppServiceProvider::register()
$this->app->bind(Mailer::class, SmtpMailer::class);
$this->app->bind(Logger::class, FileLogger::class);

// 测试时 Mock
$mockMailer = $this->createMock(Mailer::class);
$mockMailer->expects($this->once())->method('send');

$service = new OrderService($mockMailer, new NullLogger(), $this->createMock(OrderRepository::class));
$service->create(['email' => 'test@example.com']);
```

---

## 16. 实战中的模式选择

### 何时用哪种模式？

| 你的场景 | 适合的模式 |
|----------|-----------|
| 全局只需一个实例（DB/Config/Logger） | **单例** |
| 根据不同配置创建不同对象 | **工厂方法** |
| 创建一族相关的对象 | **抽象工厂** |
| 构造复杂对象（很多可选参数） | **建造者** |
| 克隆已有对象 | **原型** |
| 对接第三方库，统一接口 | **适配器** |
| 动态添加功能，不改原类 | **装饰器** |
| 延迟加载或权限控制 | **代理** |
| 事件通知（一对多） | **观察者** |
| 运行时切换算法 | **策略** |
| 层层审批/过滤 | **责任链** |
| 撤销/重做/任务队列 | **命令** |
| 固定流程，子类实现细节 | **模板方法** |
| 解除硬编码依赖 | **依赖注入** |

### 反模式警示

```php
// ❌ God Object — 一个类做了所有事
class App
{
    public function handleRequest() { /* 路由 */ }
    public function queryDb()      { /* 数据库 */ }
    public function renderView()   { /* 视图 */ }
    public function sendEmail()    { /* 邮件 */ }
    public function pay()          { /* 支付 */ }
}

// ❌ 无意义的过度设计
// 一个 20 行的 CRUD 不需要工厂+建造者+策略三层套娃
```

### 核心原则

> **模式的目的是消除变化带来的复杂度，而不是为不变的代码增加复杂度。**

---

> **文档版本**: 1.0  
> **更新日期**: 2026-05-14  
> **适用版本**: PHP 8.1+
