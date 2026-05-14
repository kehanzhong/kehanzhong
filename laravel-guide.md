# Laravel 11 框架实战指南

> 从路由到部署，涵盖 Eloquent ORM、队列、事件、广播、测试 — PHP 第一框架全栈实践

---

## 目录

1. [快速开始与项目架构](#1-快速开始与项目架构)
2. [路由、中间件与控制器](#2-路由中间件与控制器)
3. [Eloquent ORM 深入](#3-eloquent-orm-深入)
4. [Blade 模板与前端](#4-blade-模板与前端)
5. [请求验证与表单处理](#5-请求验证与表单处理)
6. [队列与任务调度](#6-队列与任务调度)
7. [事件、监听器与广播](#7-事件监听器与广播)
8. [缓存、文件存储与日志](#8-缓存文件存储与日志)
9. [测试驱动开发](#9-测试驱动开发)
10. [API 开发与 Sanctum 认证](#10-api-开发与-sanctum-认证)
11. [部署与性能优化](#11-部署与性能优化)

---

## 1. 快速开始与项目架构

### 1.1 环境要求

- PHP 8.2+
- Composer 2.x
- MySQL 8.0 / PostgreSQL 14+ / SQLite
- Redis（队列、缓存）

### 1.2 创建项目

```bash
composer create-project laravel/laravel my-app
cd my-app
php artisan serve            # 开发服务器 http://localhost:8000
php artisan key:generate     # 生成 APP_KEY
```

### 1.3 项目目录结构

```
my-app/
├── app/
│   ├── Models/              # Eloquent 模型
│   ├── Http/Controllers/    # 控制器
│   ├── Http/Middleware/     # 中间件
│   ├── Http/Requests/       # 表单请求验证
│   ├── Events/              # 事件
│   ├── Listeners/           # 监听器
│   ├── Jobs/                # 任务 Job
│   ├── Services/            # 业务逻辑层（自定义）
│   └── Providers/           # 服务提供者
├── bootstrap/               # 框架启动
├── config/                  # 配置文件
├── database/
│   ├── migrations/          # 数据库迁移
│   ├── factories/           # 模型工厂
│   └── seeders/             # 数据填充
├── public/                  # 入口 & 静态资源
├── resources/               # 视图/前端源码/CSS/JS
├── routes/                  # 路由定义
│   ├── web.php              # Web 路由（session/csrf）
│   ├── api.php              # API 路由（速率限制/token）
│   ├── channels.php         # 广播频道授权
│   └── console.php          # Artisan 命令路由
├── storage/                 # 运行时文件/日志/缓存
├── tests/                   # 测试
└── vendor/                  # 依赖
```

### 1.4 请求生命周期

```
public/index.php
  → bootstrap/app.php
    → 加载所有 ServiceProvider
    → 路由匹配 (routes/*.php)
      → 全局中间件
        → 路由中间件
          → Controller::method()
            → Response
              → 终止中间件
```

---

## 2. 路由、中间件与控制器

### 2.1 路由定义

```php
// routes/web.php

// 基础路由
Route::get('/posts', [PostController::class, 'index']);
Route::get('/posts/{post}', [PostController::class, 'show']);
Route::post('/posts', [PostController::class, 'store']);
Route::put('/posts/{post}', [PostController::class, 'update']);
Route::delete('/posts/{post}', [PostController::class, 'destroy']);

// 资源路由（自动映射 7 个方法）
Route::resource('posts', PostController::class);

// 分组
Route::prefix('admin')->middleware('auth')->group(function () {
    Route::resource('posts', Admin\PostController::class);
    Route::get('dashboard', [DashboardController::class, 'index']);
});

// 查看所有路由
// php artisan route:list
```

### 2.2 控制器

```php
<?php
namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\Request;

class PostController extends Controller
{
    public function index()
    {
        $posts = Post::with('user')->latest()->paginate(20);
        return view('posts.index', compact('posts'));
    }

    public function show(Post $post) // 隐式路由模型绑定
    {
        $post->load('comments.user'); // 延迟加载关联
        return view('posts.show', compact('post'));
    }

    public function store(StorePostRequest $request)
    {
        $post = $request->user()->posts()->create(
            $request->validated()
        );

        return redirect()->route('posts.show', $post)
            ->with('success', '文章创建成功');
    }
}
```

### 2.3 中间件

```php
// 创建中间件
// php artisan make:middleware CheckPostOwner

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class CheckPostOwner
{
    public function handle(Request $request, Closure $next)
    {
        $post = $request->route('post'); // 隐式绑定

        if ($post->user_id !== $request->user()->id) {
            abort(403, '无权操作此文章');
        }

        return $next($request);
    }
}

// 注册到 app/Http/Kernel.php
protected $routeMiddleware = [
    // ...
    'owner' => \App\Http\Middleware\CheckPostOwner::class,
];

// 使用
Route::put('/posts/{post}', [PostController::class, 'update'])
    ->middleware('owner');
```

---

## 3. Eloquent ORM 深入

### 3.1 模型定义

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Factories\HasFactory;

class Post extends Model
{
    use HasFactory;

    // 自定义表名
    protected $table = 'posts';

    // 主键类型（默认 int，UUID 时改为 string）
    protected $keyType = 'string';

    // 可批量赋值的字段
    protected $fillable = [
        'title', 'content', 'slug', 'status', 'published_at',
    ];

    // 保护字段（与 fillable 二选一）
    // protected $guarded = ['id'];

    // 类型转换
    protected function casts(): array
    {
        return [
            'published_at' => 'datetime',
            'is_pinned'    => 'boolean',
            'meta'         => 'array',     // JSON 转数组
            'price'        => 'decimal:2',
        ];
    }

    // 关联关系
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class)->latest();
    }

    public function tags()
    {
        return $this->belongsToMany(Tag::class)
            ->withTimestamps();
    }

    // 访问器（读取时转换）
    protected function title(): Attribute
    {
        return Attribute::make(
            get: fn(string $value) => ucfirst($value),
        );
    }

    // 修改器（写入时转换）
    protected function slug(): Attribute
    {
        return Attribute::make(
            set: fn(string $value) => \Str::slug($value),
        );
    }

    // 局部查询作用域
    public function scopePublished($query)
    {
        return $query->where('status', 'published')
            ->where('published_at', '<=', now());
    }

    public function scopePopular($query, int $minViews = 1000)
    {
        return $query->where('views', '>=', $minViews);
    }
}
```

### 3.2 查询构造器

```php
// 全部
$posts = Post::all();
$posts = Post::where('status', 'published')->get();

// 单条
$post = Post::find(1);                         // 按主键
$post = Post::where('slug', 'hello-world')->first();
$post = Post::firstOrCreate(                   // 查找或创建
    ['slug' => 'hello-world'],
    ['title' => 'Hello World', 'status' => 'draft']
);

// 聚合
$count = Post::where('status', 'published')->count();
$views = Post::sum('views');
$avg   = Post::whereDate('created_at', today())->avg('rating');

// 条件查询
$posts = Post::query()
    ->when(request('category'), function ($q, $cat) {
        $q->where('category', $cat);
    })
    ->when(request('search'), function ($q, $search) {
        $q->whereFullText(['title', 'content'], $search);
    })
    ->get();

// 分页
$posts = Post::paginate(20);                  // 标准分页
$posts = Post::cursorPaginate(20);            // 游标分页（大数据量）
$posts = Post::simplePaginate(20);            // 简单上一页/下一页

// 分块处理
Post::chunk(100, function ($posts) {
    foreach ($posts as $post) {
        // 处理...
    }
});

// 惰性加载（单条读取，不一次性加载全部）
foreach (Post::lazy() as $post) {
    //
}
```

### 3.3 关联查询优化

```php
// Eager Loading — 解决 N+1 问题
$posts = Post::with(['user', 'comments'])->get();
$posts = Post::with([
    'comments'          => fn($q) => $q->where('approved', true),
    'comments.user:id,name',  // 只取需要的字段
])->get();

// 预加载计数
$posts = Post::withCount('comments')
    ->orderBy('comments_count', 'desc')
    ->get();

// 延迟加载
$post = Post::find(1);
$post->load('comments.user');
$post->loadCount('comments');

// 关联查询（查询"有评论的文章"）
$posts = Post::whereHas('comments', function ($q) {
    $q->where('approved', true);
}, '>=', 3)                                   // 至少 3 条评论
    ->get();

// 关联不存在
$posts = Post::whereDoesntHave('comments')->get();
```

### 3.4 数据库迁移

```bash
php artisan make:migration create_posts_table
php artisan make:migration add_views_to_posts_table --table=posts
php artisan migrate
php artisan migrate:rollback           # 回滚最后一批
php artisan migrate:refresh            # 回滚全部 + 重新迁移
php artisan migrate:fresh              # 删除所有表 + 重新迁移（生产慎用）
```

```php
// 示例迁移
return new class extends Migration
{
    public function up(): void
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->string('title');
            $table->string('slug')->unique();
            $table->longText('content');
            $table->string('status')->default('draft');
            $table->timestamp('published_at')->nullable();
            $table->integer('views')->default(0);
            $table->timestamps();                // created_at, updated_at

            $table->index('status');
            $table->fullText(['title', 'content']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('posts');
    }
};
```

---

## 4. Blade 模板与前端

### 4.1 模板继承

```blade
<!-- resources/views/layouts/app.blade.php -->
<!DOCTYPE html>
<html>
<head>
    <title>@yield('title', config('app.name'))</title>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body>
    @include('partials.header')

    <main>
        @yield('content')
    </main>

    @include('partials.footer')

    @stack('scripts') <!-- 子模板可追加 JS -->
</body>
</html>
```

```blade
<!-- resources/views/posts/index.blade.php -->
@extends('layouts.app')

@section('title', '全部文章')

@section('content')
    @forelse($posts as $post)
        <article>
            <h2>
                <a href="{{ route('posts.show', $post) }}">
                    {{ $post->title }}
                </a>
            </h2>
            <p>{{ Str::limit($post->content, 200) }}</p>
            <time>{{ $post->published_at->format('Y-m-d') }}</time>
        </article>
    @empty
        <p>暂无文章</p>
    @endforelse

    {{ $posts->links() }}  <!-- 分页链接 -->
@endsection

@push('scripts')
    <script>console.log('Post index loaded');</script>
@endpush
```

### 4.2 Blade 常用指令

```blade
{{ $var }}                    {{-- 自动 htmlspecialchars --}}
{!! $html !!}                 {{-- 原始 HTML，XSS 风险 --}}
{{ $var ?? '默认值' }}

@if($condition) ... @elseif ... @else ... @endif
@unless($condition) ... @endunless
@isset($var) ... @endisset
@empty($var) ... @endempty

@for($i=0; $i<10; $i++) ... @endfor
@foreach($items as $item) ... @endforeach
@forelse($items as $item) ... @empty <p>空</p> @endforelse

@while($condition) ... @endwhile

@php $local = '...'; @endphp
@guest ... @endguest
@auth ... @endauth

@csrf                        {{-- CSRF 令牌 --}}
@method('PUT')               {{-- 模拟 PUT/DELETE --}}
@class(['active' => $isActive, 'disabled' => !$canEdit])

{{-- 组件 --}}
<x-alert type="success" :message="$message" />
<x-input type="email" name="email" required />
```

### 4.3 Vite 集成（前端打包）

```js
// vite.config.js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';

export default defineConfig({
    plugins: [
        laravel({
            input: ['resources/css/app.css', 'resources/js/app.js'],
            refresh: true,
        }),
    ],
});
```

```bash
npm install
npm run dev        # 开发模式（HMR）
npm run build      # 生产构建
```

---

## 5. 请求验证与表单处理

### 5.1 Form Request 验证

```bash
php artisan make:request StorePostRequest
```

```php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true; // 或检查权限
    }

    public function rules(): array
    {
        return [
            'title'       => ['required', 'string', 'max:255'],
            'slug'        => ['required', 'alpha_dash', 'unique:posts'],
            'content'     => ['required', 'string', 'min:10'],
            'category_id' => ['required', 'exists:categories,id'],
            'tags.*'      => ['exists:tags,id'],
            'image'       => ['nullable', 'image', 'max:2048'],
        ];
    }

    public function messages(): array
    {
        return [
            'title.required' => '标题不能为空',
            'slug.unique'    => '该别名已被占用',
        ];
    }

    public function attributes(): array
    {
        return [
            'category_id' => '分类',
        ];
    }
}
```

### 5.2 常用验证规则速查

```php
'email'    => 'required|email:rfc,dns|unique:users',
'password' => 'required|min:8|confirmed',
'age'      => 'required|integer|min:1|max:150',
'mobile'   => 'required|regex:/^1[3-9]\d{9}$/',
'status'   => 'required|in:draft,published,archived',
'price'    => 'required|numeric|min:0|decimal:0,2',
'start_at' => 'required|date|after:today',
'end_at'   => 'required|date|after:start_at',
'url'      => 'nullable|url:http,https',
'file'     => 'file|mimes:pdf,doc|max:10240',
// 数组
'tags'     => 'array|min:1',
'tags.*'   => 'exists:tags,id',
// 条件
'discount' => 'required_if:type,coupon|numeric|min:0|max:100',
```

### 5.3 自定义验证规则

```php
// app/Rules/ChineseIDCard.php
namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class ChineseIDCard implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (!preg_match('/^\d{17}[\dXx]$/', $value)) {
            $fail('身份证号格式不正确');
        }
    }
}

// 使用
'id_card' => ['required', new ChineseIDCard],
```

---

## 6. 队列与任务调度

### 6.1 创建与分发 Job

```bash
php artisan make:job SendWelcomeEmail
php artisan make:job GenerateReport
# 队列驱动配置在 config/queue.php
# 推荐：数据库/redis/sqs
```

```php
// app/Jobs/SendWelcomeEmail.php
namespace App\Jobs;

use App\Models\User;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;

class SendWelcomeEmail implements ShouldQueue
{
    use Queueable;

    public $tries = 3;
    public $backoff = [10, 30, 60]; // 重试间隔（秒）

    public function __construct(public User $user) {}

    public function handle(): void
    {
        // 发送邮件
        \Mail::to($this->user)->send(new \App\Mail\WelcomeMail($this->user));
    }

    public function failed(\Throwable $e): void
    {
        \Log::error('欢迎邮件发送失败', [
            'user_id' => $this->user->id,
            'error'   => $e->getMessage(),
        ]);
    }
}

// 分发 Job
SendWelcomeEmail::dispatch($user);                    // 立即入队
SendWelcomeEmail::dispatch($user)->delay(now()->addMinutes(1)); // 延迟
SendWelcomeEmail::dispatch($user)->onQueue('emails');  // 指定队列
SendWelcomeEmail::dispatch($user)->onConnection('redis'); // 指定连接

// 批量分发
$users = User::where('status', 'active')->get();
$jobs = $users->map(fn($u) => new SendWelcomeEmail($u));
\Bus::batch($jobs)->dispatch();
```

### 6.2 运行队列

```bash
php artisan queue:work            # 进程常驻（推荐生产）
php artisan queue:work --queue=emails,default
php artisan queue:work --tries=3

php artisan queue:listen          # 开发用（每次重启）

# 生产守护进程（Supervisor）
```

```ini
; /etc/supervisor/conf.d/laravel-worker.conf
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /path/to/artisan queue:work --sleep=3 --tries=3
autostart=true
autorestart=true
user=www-data
numprocs=4
```

### 6.3 任务调度（Cron）

```php
// routes/console.php
use App\Models\Post;
use Illuminate\Support\Facades\Schedule;

// 每小时清理过期 Token
Schedule::command('sanctum:prune-expired --hours=24')
    ->hourly();

// 每天备份数据库
Schedule::command('backup:run')
    ->dailyAt('02:00');

// 每周发布统计周报
Schedule::job(new GenerateWeeklyReport)
    ->weeklyOn(1, '08:00');  // 每周一 8:00

// 每分钟（高频任务）
Schedule::call(function () {
    // 检查超时未支付订单并取消
    Order::where('status', 'pending')
        ->where('created_at', '<', now()->subMinutes(30))
        ->update(['status' => 'cancelled']);
})->everyMinute();

// 服务器 Crontab（一条即可）
// * * * * * php /path/to/artisan schedule:run >> /dev/null 2>&1
```

---

## 7. 事件、监听器与广播

### 7.1 事件与监听器

```bash
php artisan make:event PostPublished
php artisan make:listener SendPostNotification --event=PostPublished
```

```php
// app/Events/PostPublished.php
namespace App\Events;

use App\Models\Post;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class PostPublished
{
    use Dispatchable, SerializesModels;

    public function __construct(public Post $post) {}
}

// app/Listeners/SendPostNotification.php
namespace App\Listeners;

use App\Events\PostPublished;

class SendPostNotification
{
    public function handle(PostPublished $event): void
    {
        // 给订阅者发通知
        $event->post->user->subscribers()->each(function ($subscriber) use ($event) {
            // 发送通知...
        });
    }
}

// 注册到 app/Providers/EventServiceProvider.php
protected $listen = [
    PostPublished::class => [
        SendPostNotification::class,
        UpdateSearchIndex::class,       // 同时更新搜索索引
    ],
];

// 分发事件
event(new PostPublished($post));
PostPublished::dispatch($post); // 自动排队（实现 ShouldQueue 时）
```

### 7.2 实时广播（WebSocket）

```php
// 广播事件需实现 ShouldBroadcast
class PostPublished implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function broadcastOn(): array
    {
        return [
            new Channel('posts'),                    // 公共频道
            new PrivateChannel('users.' . $this->post->user_id), // 私有频道
        ];
    }

    public function broadcastAs(): string
    {
        return 'post.published';
    }

    public function broadcastWith(): array
    {
        return [
            'id'    => $this->post->id,
            'title' => $this->post->title,
        ];
    }
}

// 前端接收（Laravel Echo）
// Echo.channel('posts').listen('.post.published', (e) => { ... });
```

---

## 8. 缓存、文件存储与日志

### 8.1 缓存

```php
use Illuminate\Support\Facades\Cache;

// 基础操作
Cache::put('key', 'value', now()->addMinutes(10));
$value = Cache::get('key', 'default');
Cache::forget('key');
Cache::flush();                                    // 清空

// 记住（有则返回，无则存）
$posts = Cache::remember('homepage_posts', now()->addHour(), function () {
    return Post::with('user')->latest()->take(10)->get();
});

// 永久缓存
Cache::rememberForever('site_stats', fn() => $stats);

// 原子锁
$lock = Cache::lock('order_processing_' . $orderId, 10);
if ($lock->get()) {
    try {
        // 临界代码...
    } finally {
        $lock->release();
    }
}

// 标签
Cache::tags(['posts', 'homepage'])->flush();

// 驱动：file / database / redis / memcached
// 配置在 config/cache.php
```

### 8.2 文件存储

```php
use Illuminate\Support\Facades\Storage;

// 上传文件
$path = $request->file('avatar')->store('avatars');
$path = $request->file('avatar')->store('avatars', 's3'); // 指定磁盘

// 公开可见
$path = $request->file('photo')->store('photos', 'public');

// 操作
Storage::disk('local')->put('file.txt', 'Content');
$content = Storage::get('file.txt');
Storage::delete('file.txt');
Storage::size('file.txt');
Storage::copy('old', 'new');
Storage::move('old', 'new');
$files = Storage::files('directory');

// URL
$url = Storage::url($path);                         // /storage/photos/xxx.jpg

// 临时 URL（s3 等云存储）
$url = Storage::temporaryUrl('file.pdf', now()->addMinutes(5));
```

### 8.3 日志

```php
use Illuminate\Support\Facades\Log;

Log::info('用户登录', ['user_id' => $user->id]);
Log::warning('磁盘使用率过高', ['usage' => '92%']);
Log::error('支付失败', ['order_id' => 1001, 'error' => $e->getMessage()]);
Log::debug('调试信息', $data);

// 上下文
Log::withContext(['request_id' => uniqid()]);
Log::info('收到请求');

// 自定义 Channel
// 配置在 config/logging.php，支持 daily/single/slack/papertrail 等
```

---

## 9. 测试驱动开发

### 9.1 单元测试

```bash
php artisan make:test PostTest
php artisan make:test PostTest --unit     # 不继承框架
php artisan test                          # 运行所有测试
php artisan test --filter=PostTest        # 筛选
```

```php
// tests/Feature/PostTest.php
namespace Tests\Feature;

use App\Models\Post;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;

class PostTest extends TestCase
{
    use RefreshDatabase; // 每次测试重置数据库

    public function test_authenticated_user_can_create_post(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/posts', [
                'title'   => 'Test Post',
                'slug'    => 'test-post',
                'content' => 'This is a test post content.',
            ]);

        $response->assertStatus(201)
            ->assertJsonFragment(['title' => 'Test Post']);

        $this->assertDatabaseHas('posts', [
            'slug'    => 'test-post',
            'user_id' => $user->id,
        ]);
    }

    public function test_unauthenticated_user_cannot_create_post(): void
    {
        $response = $this->postJson('/api/posts', [
            'title' => 'Test',
            'slug'  => 'test',
            'content' => '...',
        ]);

        $response->assertStatus(401);
    }

    public function test_post_title_is_required(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->postJson('/api/posts', []);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['title', 'slug', 'content']);
    }
}
```

### 9.2 模型工厂

```php
// database/factories/PostFactory.php
namespace Database\Factories;

use App\Models\Post;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

class PostFactory extends Factory
{
    protected $model = Post::class;

    public function definition(): array
    {
        return [
            'user_id'   => User::factory(),
            'title'     => fake()->sentence(),
            'slug'      => fake()->unique()->slug(),
            'content'   => fake()->paragraphs(3, true),
            'status'    => fake()->randomElement(['draft', 'published']),
            'published_at' => fake()->dateTimeBetween('-1 year'),
        ];
    }

    // State 变换
    public function published(): static
    {
        return $this->state(fn() => [
            'status'       => 'published',
            'published_at' => now(),
        ]);
    }
}

// 使用
Post::factory()->count(20)->create();
Post::factory()->published()->count(5)->create();
Post::factory()->for(User::factory()->admin())->create();
```

### 9.3 HTTP 测试

```php
$this->get('/posts')->assertStatus(200);
$this->get('/posts/999')->assertNotFound();
$this->post('/login', [...])->assertRedirect('/dashboard');

// JSON API
$this->getJson('/api/posts')
    ->assertOk()
    ->assertJsonCount(15, 'data')
    ->assertJsonStructure([
        'data' => [['id', 'title', 'created_at']],
    ]);

// 数据库断言
$this->assertDatabaseHas('users', ['email' => 'test@example.com']);
$this->assertDatabaseCount('posts', 10);
```

---

## 10. API 开发与 Sanctum 认证

### 10.1 API 路由

```php
// routes/api.php（自动带 /api 前缀，无 session/csrf）
Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('posts', Api\PostController::class);
    Route::get('/user', fn(Request $request) => $request->user());
});

// 公开路由
Route::post('/login', [AuthController::class, 'login']);
Route::post('/register', [AuthController::class, 'register']);
```

### 10.2 Sanctum Token 认证

```php
// 安装
// composer require laravel/sanctum
// php artisan sanctum:install

// app/Models/User.php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;
}

// 登录发 Token
class AuthController
{
    public function login(Request $request)
    {
        $request->validate([
            'email'    => 'required|email',
            'password' => 'required',
        ]);

        if (!Auth::attempt($request->only('email', 'password'))) {
            return response()->json(['message' => '登录失败'], 401);
        }

        $user = $request->user();
        $token = $user->createToken('api-token', ['read'])->plainTextToken;

        // 可指定过期时间（需单独实现）和权限范围
        // $token = $user->createToken('api-token', ['*'])->plainTextToken;

        return response()->json([
            'token' => $token,
            'user'  => $user,
        ]);
    }

    public function logout(Request $request)
    {
        $request->user()->currentAccessToken()->delete();
        return response()->json(['message' => '已登出']);
    }
}
```

### 10.3 API 资源转换

```bash
php artisan make:resource PostResource
```

```php
// app/Http/Resources/PostResource.php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'         => $this->id,
            'title'      => $this->title,
            'slug'       => $this->slug,
            'excerpt'    => Str::limit($this->content, 200),
            'author'     => [
                'name'   => $this->user->name,
                'avatar' => $this->user->avatar_url,
            ],
            'tags'       => $this->tags->pluck('name'),
            'comments_count' => $this->whenCounted('comments'),
            'created_at' => $this->created_at->toIso8601String(),
            'updated_at' => $this->updated_at->toIso8601String(),
        ];
    }
}

// 控制器中使用
class PostController
{
    public function index()
    {
        return PostResource::collection(
            Post::with('user', 'tags')->withCount('comments')->paginate(20)
        );
    }

    public function show(Post $post)
    {
        return new PostResource($post->load('user', 'tags'));
    }
}
```

### 10.4 速率限制

```php
// config/app/Http/Kernel.php 默认已配置
// api 路由：每分钟 60 次（默认）

// 自定义
Route::middleware('throttle:30,1')->group(function () {
    Route::post('/verify-code', ...); // 1 分钟 30 次
});

// 也可用 Redis 限流器
use Illuminate\Support\Facades\RateLimiter;

RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});
```

---

## 11. 部署与性能优化

### 11.1 生产环境优化

```bash
# 优化
php artisan optimize           # 缓存配置/路由/事件
php artisan config:cache       # 缓存配置
php artisan route:cache        # 缓存路由
php artisan event:cache        # 缓存事件
php artisan view:cache         # 缓存视图

# 取消缓存（开发时）
php artisan optimize:clear

# Composer
composer install --optimize-autoloader --no-dev
```

### 11.2 Nginx 配置

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/my-app/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

### 11.3 部署脚本

```bash
#!/bin/bash
# deploy.sh
set -e

cd /var/www/my-app
git pull origin main

composer install --no-dev --optimize-autoloader
npm ci && npm run build

php artisan migrate --force
php artisan optimize
php artisan queue:restart
```

### 11.4 性能检查清单

| 检查项 | 命令/方法 |
|--------|-----------|
| 配置缓存 | `php artisan config:cache` |
| 路由缓存 | `php artisan route:cache` |
| 视图缓存 | `php artisan view:cache` |
| OPcache | `php -i | grep opcache.enable` |
| N+1 查询 | Laravel Debugbar / Telescope |
| 队列驱动 | Redis（非 database） |
| 缓存驱动 | Redis（非 file） |
| 会话驱动 | Redis / database |
| 静态资源 CDN | 配置 `ASSET_URL` |
| HTTP/2 | Nginx 配置 |

---

## 附录：常用 Artisan 命令速查

```bash
# 生成
php artisan make:model Post --all         # 同时生成 migration/factory/seeder/controller
php artisan make:model Post -mf           # migration + factory
php artisan make:controller PostController --resource
php artisan make:request StorePostRequest
php artisan make:resource PostResource
php artisan make:job ProcessPayment
php artisan make:event OrderShipped
php artisan make:listener SendNotification
php artisan make:command SendReport

# 数据库
php artisan migrate
php artisan migrate:fresh --seed
php artisan db:seed
php artisan db:show                         # 显示数据库信息

# 路由
php artisan route:list
php artisan route:list --path=api

# 缓存
php artisan cache:clear
php artisan config:clear
php artisan view:clear
php artisan optimize

# 队列
php artisan queue:work
php artisan queue:retry all                 # 重试全部失败任务
php artisan queue:failed                    # 查看失败任务

# Tinker（交互式 REPL）
php artisan tinker
# >> App\Models\User::count()
# >> App\Models\Post::where('status', 'published')->count()
```

---

> **文档版本**: 1.0  
> **更新日期**: 2026-05-14  
> **适用版本**: Laravel 11.x / PHP 8.2+
