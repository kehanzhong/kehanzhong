# 操作系统核心概念与 PHP 高性能应用

> 进程、线程、协程 + Swoole + Workerman 实战

---

## 目录

1. [进程（Process）](#1-进程process)
2. [线程（Thread）](#2-线程thread)
3. [协程（Coroutine）](#3-协程coroutine)
4. [进程 vs 线程 vs 协程](#4-进程-vs-线程-vs-协程)
5. [上下文切换](#5-上下文切换)
6. [I/O 模型](#6-io-模型)
7. [Swoole 实战](#7-swoole-实战)
8. [Workerman 实战](#8-workerman-实战)
9. [Swoole vs Workerman](#9-swoole-vs-workerman)
10. [常驻内存最佳实践](#10-常驻内存最佳实践)
11. [实战项目](#11-实战项目)
12. [性能对比与压测](#12-性能对比与压测)

---

## 1. 进程（Process）

### 1.1 什么是进程

```
进程 = 正在运行的程序实例

操作系统分配资源的基本单位：
- 独立的内存空间（代码段、数据段、堆、栈）
- 文件描述符表
- 进程 ID（PID）
- 信号处理表
- CPU 时间片

每个 PHP-FPM worker 就是一个独立的进程
```

```
┌─────────────────────────────────┐
│         进程内存布局              │
├─────────────────────────────────┤
│  高地址                          │
│  ┌──────────┐                   │
│  │  栈 (Stack) │ ← 函数调用、局部变量│
│  ├──────────┤                   │
│  │     ↓     │                   │
│  │    ...    │                   │
│  │     ↑     │                   │
│  ├──────────┤                   │
│  │  堆 (Heap)  │ ← 动态分配(malloc) │
│  ├──────────┤                   │
│  │ 数据段     │ ← 全局/静态变量     │
│  ├──────────┤                   │
│  │ 代码段     │ ← 程序指令          │
│  └──────────┘                   │
│  低地址                          │
└─────────────────────────────────┘
```

### 1.2 进程状态

```
创建 (New) ──→ 就绪 (Ready) ──→ 运行 (Running) ──→ 终止 (Terminated)
                   ↑                │
                   │      ┌─────────┤
                   │      ▼         ▼
                   └── 阻塞 (Blocked)  超时 → 就绪
                        (等待 I/O/锁)

阻塞 vs 挂起：
- 阻塞：主动等待（进程还在内存）
- 挂起：被操作系统换出到磁盘（Swap）
```

### 1.3 PHP 多进程

```php
/**
 * pcntl_fork 创建子进程
 *
 * fork 后父子进程拥有独立的内存空间（写时复制）
 */
$pid = pcntl_fork();

if ($pid == -1) {
    die('fork 失败');
} elseif ($pid == 0) {
    // ═══ 子进程 ═══
    echo "子进程 PID: " . posix_getpid() . "\n";
    
    // 子进程可以 exec() 换成别的程序
    // pcntl_exec('/usr/bin/php', ['worker.php']);
    
    exit(0);  // 子进程必须 exit，否则会继续执行
} else {
    // ═══ 父进程 ═══
    echo "父进程 PID: " . posix_getpid() . ", 子进程 PID: {$pid}\n";
    
    // 等待子进程结束，防止僵尸进程
    pcntl_wait($status);
    echo "子进程退出状态: {$status}\n";
}
```

### 1.4 僵尸进程与孤儿进程

```php
// 僵尸进程：子进程退出了但父进程没有 wait → 进程表残留
// 孤儿进程：父进程先退出 → 子进程被 init(PID=1) 收养

// ✅ 防止僵尸进程的三种方法
// 方法1：父进程 wait
pcntl_wait($status);

// 方法2：忽略 SIGCHLD 信号
pcntl_signal(SIGCHLD, SIG_IGN);

// 方法3：双重 fork（孙子进程被 init 收养）
$pid = pcntl_fork();
if ($pid > 0) {
    pcntl_wait($status);  // 父进程立即回收
    return;
}
// 子进程再 fork
$pid2 = pcntl_fork();
if ($pid2 > 0) {
    exit(0);  // 子进程退出，孙子进程变孤儿被 init 收养
}
// 孙子进程做事...
```

### 1.5 进程间通信（IPC）

```php
// ═══ 1. 管道（Pipe） ═══
// 单向数据流，父子进程间

$fds = [];
// stream_socket_pair 创建双向通道
stream_socket_pair(STREAM_PF_UNIX, STREAM_SOCK_STREAM, STREAM_IPPROTO_IP, $fds);

$pid = pcntl_fork();
if ($pid == 0) {
    fclose($fds[0]);
    fwrite($fds[1], "子进程消息");
    $msg = fread($fds[1], 1024);  // 阻塞等父进程回复
    fclose($fds[1]);
    exit(0);
}

fclose($fds[1]);
$msg = fread($fds[0], 1024);  // 收到"子进程消息"
fwrite($fds[0], "父进程已收到");
pcntl_wait($status);
fclose($fds[0]);


// ═══ 2. 共享内存（shmop） ═══
// 最快但需要自己处理并发

$key = ftok(__FILE__, 'a');
$shmId = shmop_open($key, 'c', 0644, 1024);
shmop_write($shmId, '共享数据', 0);
$data = shmop_read($shmId, 0, shmop_size($shmId));
shmop_close($shmId);


// ═══ 3. 消息队列 ═══
$key = ftok(__FILE__, 'b');
$queue = msg_get_queue($key);

// 父进程发送
msg_send($queue, 1, "任务数据");

// 子进程接收
msg_receive($queue, 0, $msgType, 1024, $data);


// ═══ 4. 信号 ═══
pcntl_signal(SIGUSR1, function ($sig) {
    echo "收到自定义信号\n";
});
posix_kill($childPid, SIGUSR1);  // 给子进程发信号


// ═══ 5. Unix Socket ═══
// 见 Swoole/Workerman 章节
```

---

## 2. 线程（Thread）

### 2.1 什么是线程

```
线程 = 进程内的执行流（轻量级进程）

进程：
┌─────────────────────────────────┐
│  代码段 数据段 堆 文件描述符     │ ← 共享
├──────┬──────┬──────┬───────────┤
│ 线程1 │ 线程2 │ 线程3 │ 线程4     │
│ 栈1   │ 栈2   │ 栈3   │ 栈4      │ ← 独立
│ 寄存器 │ 寄存器 │ 寄存器 │ 寄存器   │
└──────┴──────┴──────┴───────────┘

线程共享：
  - 代码段、数据段、堆内存
  - 文件描述符
  - 信号处理

线程独有：
  - 栈（局部变量、函数调用链）
  - 寄存器
  - 线程 ID
  - errno
```

### 2.2 PHP 线程（pthreads → parallel）

```php
// pthreads 已废弃（PHP 7.4 移除），PHP 8+ 用 parallel 扩展
// 注意：PHP 线程只适合 CLI 模式

// parallel 扩展安装
// pecl install parallel

use parallel\Runtime;
use parallel\Future;

// ═══ 创建线程执行任务 ═══
$runtime = new Runtime();

$future = $runtime->run(function () {
    // 这里是独立线程
    sleep(2);
    return "线程结果";
});

echo "主线程继续执行...\n";
$result = $future->value();  // 阻塞等结果
echo "获取结果: {$result}\n";

// ═══ 多线程并行 ═══
$futures = [];
for ($i = 0; $i < 4; $i++) {
    $runtime = new Runtime();
    $futures[] = $runtime->run(function ($i) {
        sleep(1);
        return "线程 {$i} 完成";
    }, [$i]);
}

foreach ($futures as $future) {
    echo $future->value() . "\n";
}
// 总耗时 ≈ 1秒（非 4秒）
```

### 2.3 线程安全问题

```php
// ⚠️ 多线程共享数据需要加锁

$runtime = new Runtime();
$future = $runtime->run(function () {
    // parallel 中每个线程有独立内存
    // 不能直接共享变量，需要通过 Channel 通信

    $channel = \parallel\Channel::make('tasks', \parallel\Channel::Infinite);

    return function () use ($channel) {
        $channel->send('data');
    };
});

// ✅ 用 Channel 通信，不用共享内存
// parallel 默认隔离，反而是最安全的设计
```

---

## 3. 协程（Coroutine）

### 3.1 什么是协程

```
协程 = 用户态的轻量级"线程"，由程序自己调度，不是操作系统调度

关键：协程不是线程！协程是函数级的中断与恢复。

┌─────────────────────────────────────┐
│              进程                    │
├─────────────────────────────────────┤
│              线程 1                  │
├─────────────────────────────────────┤
│  ┌─ 协程A ─┐ ┌─ 协程B ─┐ ┌─ 协程C ─┐ │
│  │ 代码1   │ │ 代码1   │ │ 代码1   │ │
│  │ 协程切换 │ │ 协程切换 │ │ await   │ │
│  │ 代码2   │ │ 代码2   │ │ 代码2   │ │
│  │ yield   │ │ yield   │ │ 协程切换 │ │
│  └─────────┘ └─────────┘ └─────────┘ │
└─────────────────────────────────────┘

协程切换由程序控制：
- 在 I/O 等待时主动让出（yield）
- 切换成本极低（纳秒级，不涉及系统调用）
- 一个线程可以运行成千上万个协程
```

### 3.2 PHP Generator（协程基础）

```php
// Generator 是 PHP 协程的基石

function myCoroutine()
{
    echo "协程开始\n";
    $data = yield '第一步结果';  // 暂停，返回值给调用者
    echo "收到数据: {$data}\n";
    yield '第二步结果';
    echo "协程结束\n";
}

$gen = myCoroutine();

// 启动协程
$step1 = $gen->current();     // "协程开始" → "第一步结果"
echo "拿到: {$step1}\n";

// 传递数据并恢复
$step2 = $gen->send('外部数据');  // "收到数据: 外部数据" → "第二步结果"
echo "拿到: {$step2}\n";

$gen->next();  // "协程结束"
```

### 3.3 Swoole 协程（Go 风格）

```php
// Swoole 4.0+ 协程：自动调度，不需要手动 yield

use Swoole\Coroutine;

// ═══ 基础协程 ═══
Coroutine\run(function () {
    // 这个闭包在协程中执行
    echo "协程1开始\n";
    
    // 创建子协程
    Coroutine::create(function () {
        echo "协程2开始\n";
        sleep(2);  // ⚠️ 这会阻塞整个进程！
        echo "协程2结束\n";
    });
    
    echo "协程1继续\n";
});

// ═══ 正确的协程 sleep ═══
Coroutine\run(function () {
    $start = microtime(true);

    // 并行执行 3 个协程（都会异步 sleep）
    Coroutine::create(function () {
        Coroutine::sleep(2);  // ✅ 不阻塞，让出给其他协程
        echo "协程A完成\n";
    });
    Coroutine::create(function () {
        Coroutine::sleep(1);
        echo "协程B完成\n";
    });
    Coroutine::create(function () {
        Coroutine::sleep(3);
        echo "协程C完成\n";
    });

    Coroutine::sleep(3.1);  // 等它们都完成
    echo "总耗时: " . round(microtime(true) - $start, 2) . "秒\n";
    // 输出: ≈3.1秒（不是 6秒！）
});

// ═══ 协程并发 HTTP ═══
Coroutine\run(function () {
    $start = microtime(true);
    $results = [];

    for ($i = 1; $i <= 5; $i++) {
        Coroutine::create(function () use ($i, &$results) {
            $client = new \Swoole\Coroutine\Http\Client('api.example.com', 443, true);
            $client->get("/data/{$i}");
            $results[$i] = $client->body;
            $client->close();
        });
    }

    echo "总耗时: " . round(microtime(true) - $start, 2) . "秒\n";
    // 5个HTTP请求并发 ≈ 最慢那一个的时间
});
```

### 3.4 协程调度原理

```
Swoole 协程调度器：

Event Loop（单线程）
   │
   ├─ 协程A：执行 → 遇到 I/O → 注册回调 → yield
   ├─ 协程B：执行 → 遇到 I/O → 注册回调 → yield
   ├─ 协程C：执行 → 完毕
   │
   ├─ [I/O 完成通知] → 协程A 被唤醒 → 继续执行
   ├─ [I/O 完成通知] → 协程B 被唤醒 → 继续执行

关键：协程切换不经过操作系统，CPU 不会"上下文切换"
```

---

## 4. 进程 vs 线程 vs 协程

| 维度 | 进程 | 线程 | 协程 |
|------|------|------|------|
| **调度者** | 操作系统 | 操作系统 | 用户程序 |
| **内存开销** | MB 级 | MB 级（栈 8MB） | KB 级（栈 4KB） |
| **创建速度** | 慢（ms 级） | 中（μs 级） | 快（ns 级） |
| **切换成本** | 高（系统调用+TLB刷新） | 中（系统调用） | 极低（函数调用） |
| **通信方式** | IPC（管道/共享内存） | 共享内存（需要锁） | Channel（无锁） |
| **数据隔离** | 强隔离 | 弱隔离 | 隔离（Channel） |
| **并发能力** | 几十~几百 | 几百~几千 | 数万~百万 |
| **PHP 支持** | pcntl_fork | parallel 扩展 | Swoole/Fiber(PHP 8.1+) |
| **崩溃影响** | 只影响自己 | 可能拖垮进程 | 通常可被捕获 |

```
选择建议：

CPU 密集型：多进程（每个进程用满一个核）
I/O 密集型：协程（在等待 I/O 时做其他事）
混合型：    多进程 + 每进程内协程

PHP 典型方案：
- Web 请求：Swoole Coroutine（I/O 密集）
- 后台任务：进程池 + 协程
- 实时通信：Swoole WebSocket + 协程
```

---

## 5. 上下文切换

### 5.1 进程上下文切换

```
进程切换发生了什么：
1. 保存当前进程的寄存器、程序计数器
2. 切换虚拟内存页表（TLB 刷新）
3. 切换内核栈
4. 加载新进程的上下文
5. 耗时：1-10 微秒 / 次

高频切换 → CPU 大量时间浪费在「切换」而非「工作」
```

### 5.2 线程切换

```
线程切换（同进程内）：
- 不需要切换页表（共享地址空间）
- 但仍需要系统调用和内核态切换
- 耗时：~1 微秒

线程切换（不同进程）：
- 和进程切换一样昂贵
```

### 5.3 协程切换

```
协程切换：
- 完全在用户态完成
- 只保存/恢复少量寄存器和栈指针
- 没有系统调用，没有 TLB 刷新
- 耗时：~10-100 纳秒
```

---

## 6. I/O 模型

### 6.1 五种 I/O 模型

```
1. 阻塞 I/O（BIO）
   ┌───────┐     ┌───────┐     ┌───────┐
   │recvfrom│────→│ 内核    │────→│ 数据   │
   │ 阻塞等待 │     │ 准备数据 │     │ 就绪   │
   └───────┘     └───────┘     └───┬───┘
                                   │
   ┌───────┐     ┌───────┐         │
   │复制完成│←────│ 内核复制│←────────┘
   │ 返回   │     │ 到用户态 │
   └───────┘     └───────┘
   
   PHP-FPM 就是这种模式：每个请求阻塞等待

2. 非阻塞 I/O（NIO）
   ┌───────┐     ┌───────┐
   │recvfrom│────→│ 无数据  │  → 返回 EAGAIN → 再试 → 再试...
   │ 轮询   │     │        │
   └───────┘     └───────┘
   浪费 CPU 做轮询

3. I/O 多路复用（select/poll/epoll）⭐
   ┌───────┐     ┌───────────┐
   │ select│────→│ 等待多个fd  │  → 有数据就绪 → 通知
   │/epoll │     │ 中的任意一个│
   └───────┘     └───────────┘
   
   Swoole/Workerman 的核心：一个线程管理万个连接

4. 信号驱动 I/O
   内核在数据就绪时发 SIGIO 信号通知

5. 异步 I/O（AIO）
   数据全部复制完成后再通知，完全无阻塞
```

### 6.2 epoll 原理

```
epoll 是 Linux 最高效的 I/O 多路复用机制：

传统 select/poll：
  每次调用都要传整个 fd 列表 → O(n) 扫描

epoll：
  1. epoll_create：在内核创建事件表
  2. epoll_ctl：注册 fd（只做一次）
  3. epoll_wait：只返回就绪的 fd → O(1)

  使用红黑树 + 就绪链表
  适合成千上万个连接
```

### 6.3 PHP 的 I/O 多路复用

```php
// 原生 socket_select（底层选 epoll）
$read = [$socket1, $socket2, $socket3];
$write = null;
$except = null;

// 阻塞等任意一个 fd 就绪
if (socket_select($read, $write, $except, 5) > 0) {
    foreach ($read as $socket) {
        $data = socket_read($socket, 1024);
        // 处理...
    }
}

// Swoole 和 Workerman 内部都用 epoll
// 开发者不需要手动处理这些
```

---

## 7. Swoole 实战

### 7.1 安装

```bash
# pecl 安装
pecl install swoole

# 编译安装
git clone https://github.com/swoole/swoole-src.git
cd swoole-src
phpize
./configure --enable-openssl --enable-swoole-curl
make -j$(nproc)
make install

# 查看扩展
php -m | grep swoole
```

### 7.2 HTTP 服务器

```php
// http_server.php
$http = new Swoole\Http\Server('0.0.0.0', 9501);

// 工作进程启动
$http->on('WorkerStart', function ($server, $workerId) {
    // 每个 Worker 进程启动时执行
    // 可以用来初始化连接池、加载配置
    echo "Worker #{$workerId} 启动\n";
});

// 处理请求（自动协程化）
$http->on('Request', function ($request, $response) {
    // 并发调用多个 API
    $userData = null;
    $orderData = null;

    Swoole\Coroutine::create(function () use (&$userData) {
        $client = new Swoole\Coroutine\Http\Client('user-api.internal', 80);
        $client->get('/user/123');
        $userData = json_decode($client->body, true);
        $client->close();
    });

    Swoole\Coroutine::create(function () use (&$orderData) {
        $db = new Swoole\Coroutine\MySQL();
        $db->connect([
            'host'     => '127.0.0.1',
            'user'     => 'root',
            'password' => 'pass',
            'database' => 'mydb',
        ]);
        $orderData = $db->query('SELECT * FROM orders WHERE user_id = 123');
        $db->close();
    });

    // 等两个协程完成（这里应该用 WaitGroup）
    Swoole\Coroutine::sleep(0.1);

    // 响应
    $response->header('Content-Type', 'application/json');
    $response->end(json_encode([
        'code'   => 0,
        'data'   => [
            'user'   => $userData,
            'orders' => $orderData,
        ],
    ]));
});

$http->set([
    'worker_num'      => 4,     // Worker 进程数（CPU核数 × 1-2）
    'max_request'     => 10000, // 处理10000个请求后重启（防内存泄漏）
    'daemonize'       => false, // 开发环境不守护
    'log_file'        => '/var/log/swoole.log',
    'enable_coroutine' => true, // 自动协程化
]);

$http->start();
```

### 7.3 TCP 服务器

```php
// tcp_server.php
$server = new Swoole\Server('0.0.0.0', 9502, SWOOLE_PROCESS, SWOOLE_SOCK_TCP);

$server->set([
    'worker_num'          => 4,
    'task_worker_num'     => 2,   // Task 进程数
    'task_enable_coroutine' => true,
    'heartbeat_check_interval' => 60,  // 心跳检测间隔
    'heartbeat_idle_time'      => 300, // 300秒没数据就断开
    'open_eof_split'      => true,      // 按结束符分包
    'package_eof'         => "\r\n",     // 结束符
    'reload_async'        => true,       // 异步安全重载
]);

// 连接事件
$server->on('Connect', function ($server, $fd) {
    echo "客户端 #{$fd} 连接\n";
});

// 接收数据
$server->on('Receive', function ($server, $fd, $reactorId, $data) {
    $data = trim($data);
    echo "收到 #{$fd}: {$data}\n";

    // 投递异步任务（耗时操作）
    $server->task([
        'fd'   => $fd,
        'data' => $data,
    ]);

    $server->send($fd, "已收到: {$data}\n");
});

// 异步任务处理
$server->on('Task', function ($server, $taskId, $srcWorkerId, $data) {
    // 这里可以做耗时操作（发邮件、写日志等）
    echo "处理任务 #{$taskId}: {$data['data']}\n";

    // 可以在这里使用协程
    $client = new Swoole\Coroutine\Http\Client('log.internal', 80);
    $client->post('/log', ['message' => $data['data']]);
    $client->close();

    return "任务完成";
});

// 任务完成回调
$server->on('Finish', function ($server, $taskId, $result) {
    echo "任务 #{$taskId} 完成: {$result}\n";
});

// 关闭连接
$server->on('Close', function ($server, $fd) {
    echo "客户端 #{$fd} 关闭\n";
});

$server->start();
```

### 7.4 WebSocket 服务器

```php
// websocket_server.php
$ws = new Swoole\WebSocket\Server('0.0.0.0', 9503);

$ws->set([
    'worker_num' => 2,
    'enable_coroutine' => true,
]);

// 连接池（在 Worker 进程中维护，所有协程共享）
$connections = [];

$ws->on('Open', function ($server, $request) use (&$connections) {
    $userId = $request->get['user_id'] ?? 0;
    if ($userId) {
        $connections[$userId] = $request->fd;
        echo "用户 {$userId} 上线 (fd: {$request->fd})\n";
    }
});

$ws->on('Message', function ($server, $frame) use (&$connections) {
    $data = json_decode($frame->data, true);

    switch ($data['type'] ?? '') {
        case 'chat':
            // 私聊：发给指定用户
            $targetFd = $connections[$data['to']] ?? null;
            if ($targetFd) {
                $server->push($targetFd, json_encode([
                    'from'    => $data['from'],
                    'message' => $data['message'],
                    'time'    => date('H:i:s'),
                ]));
            }
            break;

        case 'broadcast':
            // 广播：发给所有连接
            foreach ($server->connections as $fd) {
                $server->push($fd, json_encode([
                    'from'    => '系统',
                    'message' => $data['message'],
                ]));
            }
            break;

        case 'ping':
            $server->push($frame->fd, json_encode(['type' => 'pong']));
            break;
    }
});

$ws->on('Close', function ($server, $fd) use (&$connections) {
    $connections = array_filter($connections, fn($f) => $f !== $fd);
    echo "fd: {$fd} 断开\n";
});

$ws->start();
```

### 7.5 协程 WaitGroup

```php
use Swoole\Coroutine\WaitGroup;
use Swoole\Coroutine;

Coroutine\run(function () {
    $wg = new WaitGroup();
    $results = new Swoole\ArrayObject();

    // 添加 3 个并发任务
    $wg->add(3);

    Swoole\Coroutine::create(function () use ($wg, $results) {
        $client = new Swoole\Coroutine\Http\Client('api1.com', 443, true);
        $client->get('/data');
        $results['api1'] = $client->body;
        $client->close();
        $wg->done();
    });

    Swoole\Coroutine::create(function () use ($wg, $results) {
        Coroutine::sleep(1.5);
        $results['sleep'] = 'done';
        $wg->done();
    });

    Swoole\Coroutine::create(function () use ($wg, $results) {
        $db = new Swoole\Coroutine\MySQL();
        $db->connect(['host' => '127.0.0.1', 'user' => 'root', 'password' => 'pass', 'database' => 'test']);
        $results['db'] = $db->query('SELECT NOW()');
        $db->close();
        $wg->done();
    });

    // 等待全部完成
    $wg->wait();

    var_dump($results->getArrayCopy());
    // 3个任务并发执行，总耗时 ≈ 最慢那个的时间
});
```

### 7.6 协程 Channel

```php
use Swoole\Coroutine\Channel;
use Swoole\Coroutine;

Coroutine\run(function () {
    $channel = new Channel(2);  // 容量 2

    // ═══ 生产者 ═══
    Coroutine::create(function () use ($channel) {
        for ($i = 1; $i <= 5; $i++) {
            Coroutine::sleep(0.5);
            $channel->push("数据 {$i}");
            echo "生产: 数据 {$i}\n";
        }
        $channel->close();
    });

    // ═══ 消费者 ═══
    Coroutine::create(function () use ($channel) {
        while (true) {
            $data = $channel->pop(2);  // 等最多2秒
            if ($data === false) {
                echo "消费结束\n";
                break;
            }
            echo "消费: {$data}\n";
            Coroutine::sleep(0.8);  // 消费比生产慢
        }
    });
});
```

### 7.7 协程连接池

```php
final class MySQLPool
{
    private Swoole\Coroutine\Channel $pool;
    private array $config;

    public function __construct(array $config, int $size = 20)
    {
        $this->config = $config;
        $this->pool = new Channel($size);

        for ($i = 0; $i < $size; $i++) {
            $this->pool->push($this->createConnection());
        }
    }

    private function createConnection(): Swoole\Coroutine\MySQL
    {
        $db = new Swoole\Coroutine\MySQL();
        $db->connect($this->config);
        return $db;
    }

    public function get(float $timeout = 5): Swoole\Coroutine\MySQL
    {
        $db = $this->pool->pop($timeout);
        if ($db === false) {
            throw new \RuntimeException('获取数据库连接超时');
        }
        return $db;
    }

    public function put(Swoole\Coroutine\MySQL $db): void
    {
        if (!$db->connected) {
            $db = $this->createConnection();
        }
        $this->pool->push($db);
    }

    public function exec(callable $fn): mixed
    {
        $db = $this->get();
        try {
            return $fn($db);
        } finally {
            $this->put($db);
        }
    }
}

// 使用（在 WorkerStart 中创建一次）
$pool = new MySQLPool([
    'host'     => '127.0.0.1',
    'user'     => 'root',
    'password' => 'pass',
    'database' => 'mydb',
], 20);

// 协程中安全使用
Coroutine::create(function () use ($pool) {
    $result = $pool->exec(function ($db) {
        return $db->query('SELECT * FROM users WHERE id = 1');
    });
    // 连接自动归还
});
```

### 7.8 定时器

```php
// ═══ 毫秒级定时器 ═══
$timerId = Swoole\Timer::tick(1000, function ($timerId) {
    echo "每秒执行一次\n";
});

// 5秒后清除
Swoole\Timer::after(5000, function () use ($timerId) {
    Swoole\Timer::clear($timerId);
    echo "定时器已清除\n";
});

// ═══ 延迟执行 ═══
Swoole\Timer::after(3000, function () {
    echo "3秒后执行\n";
});
```

---

## 8. Workerman 实战

### 8.1 安装

```bash
composer require workerman/workerman
```

### 8.2 HTTP 服务器

```php
// http_server.php
require_once __DIR__ . '/vendor/autoload.php';

use Workerman\Worker;
use Workerman\Protocols\Http\Request;
use Workerman\Protocols\Http\Response;

$http = new Worker('http://0.0.0.0:8080');

// 进程数
$http->count = 4;
$http->name = 'MyHttpServer';

// 每个进程启动时
$http->onWorkerStart = function ($worker) {
    echo "Worker #{$worker->id} 启动\n";

    // 初始化数据库连接（Workerman 是多进程，每进程独立）
    global $db;
    $db = new PDO('mysql:host=127.0.0.1;dbname=mydb', 'root', 'pass');
};

// 处理请求
$http->onMessage = function ($connection, Request $request) {
    $uri = $request->uri();
    $method = $request->method();

    // 路由
    switch (true) {
        case $uri === '/api/users' && $method === 'GET':
            global $db;
            $users = $db->query('SELECT id, name FROM users LIMIT 10')
                         ->fetchAll(PDO::FETCH_ASSOC);

            $response = new Response(200, ['Content-Type' => 'application/json'], json_encode($users));
            break;

        case $uri === '/api/status':
            $response = new Response(200, ['Content-Type' => 'application/json'], json_encode([
                'status' => 'ok',
                'time'   => date('Y-m-d H:i:s'),
                'pid'    => getmypid(),
            ]));
            break;

        default:
            $response = new Response(404, [], 'Not Found');
    }

    $connection->send($response);
};

Worker::runAll();
```

### 8.3 WebSocket 服务器

```php
// ws_server.php
use Workerman\Worker;

$ws = new Worker('websocket://0.0.0.0:8282');
$ws->count = 2;
$ws->name = 'ChatServer';

// 存储连接（userId => connection）
$connections = [];

$ws->onConnect = function ($connection) {
    echo "新连接: {$connection->id}\n";
};

$ws->onMessage = function ($connection, $data) use (&$connections) {
    $message = json_decode($data, true);

    switch ($message['type']) {
        case 'login':
            // 用户登录绑定
            $connections[$message['user_id']] = $connection;
            $connection->userId = $message['user_id'];

            $connection->send(json_encode([
                'type' => 'login_success',
                'online' => count($connections),
            ]));

            // 广播上线通知
            foreach ($connections as $conn) {
                if ($conn->id !== $connection->id) {
                    $conn->send(json_encode([
                        'type' => 'user_online',
                        'user_id' => $message['user_id'],
                    ]));
                }
            }
            break;

        case 'chat':
            // 私聊
            $targetConn = $connections[$message['to']] ?? null;
            if ($targetConn) {
                $targetConn->send(json_encode([
                    'type' => 'chat',
                    'from' => $connection->userId,
                    'message' => $message['message'],
                    'time'  => date('H:i:s'),
                ]));
            }
            break;

        case 'ping':
            $connection->send(json_encode(['type' => 'pong']));
            break;
    }
};

$ws->onClose = function ($connection) use (&$connections) {
    if (isset($connection->userId)) {
        unset($connections[$connection->userId]);

        // 广播下线通知
        foreach ($connections as $conn) {
            $conn->send(json_encode([
                'type' => 'user_offline',
                'user_id' => $connection->userId,
            ]));
        }
    }
    echo "连接关闭: {$connection->id}\n";
};

Worker::runAll();
```

### 8.4 TCP 服务器

```php
// tcp_server.php
use Workerman\Worker;

$tcp = new Worker('tcp://0.0.0.0:9000');
$tcp->count = 4;

$tcp->onMessage = function ($connection, $data) {
    $data = trim($data);
    echo "收到: {$data}\n";

    // 业务处理
    $result = match (true) {
        str_starts_with($data, 'PING') => '+PONG',
        str_starts_with($data, 'TIME') => date('Y-m-d H:i:s'),
        str_starts_with($data, 'ECHO ') => substr($data, 5),
        default => '-ERR 未知命令',
    };

    $connection->send("{$result}\r\n");
};

Worker::runAll();
```

### 8.5 定时器

```php
use Workerman\Worker;
use Workerman\Timer;

$worker = new Worker();
$worker->count = 1;  // 定时任务只需要1个进程

$worker->onWorkerStart = function () {
    // 每秒执行
    Timer::add(1, function () {
        echo "定时任务: " . date('Y-m-d H:i:s') . "\n";
    });

    // 一次性定时（5秒后执行）
    Timer::add(5, function () {
        echo "一次性任务\n";
    }, [], false);  // false = 不重复
};

Worker::runAll();
```

---

## 9. Swoole vs Workerman

| 维度 | Swoole | Workerman |
|------|--------|-----------|
| **语言** | PHP + C 扩展 | 纯 PHP |
| **安装** | pecl install（需编译） | composer require |
| **协程** | ✅ 原生协程（自动） | ❌ 无协程（纯多进程） |
| **并发模型** | 多进程 + 协程 | 多进程 |
| **HTTP 服务器** | ✅ built-in | ✅ built-in |
| **WebSocket** | ✅ | ✅ |
| **TCP/UDP** | ✅ | ✅ |
| **MySQL 连接池** | ✅ 协程连接池 | ❌ 需自己实现 |
| **性能** | 极高 | 很高 |
| **学习曲线** | 陡峭 | 平缓 |
| **社区生态** | Hyperf/EasySwoole | webman/GatewayWorker |
| **PHP 版本** | 7.2+ | 7.0+ |
| **调试** | 较难（C扩展级） | 容易（纯PHP） |

```
选型建议：

选 Swoole：
  - 需要高并发（万级以上QPS）
  - 需要协程处理大量 I/O
  - 有基础愿意学习
  - 项目用 Hyperf/EasySwoole 框架

选 Workerman：
  - 快速上手
  - 纯 PHP 无编译依赖
  - 并发要求适中（千级QPS足够）
  - 项目用 webman 框架

Swoole→高性能高并发 I/O密集
Workerman→快速开发 稳定可靠 纯PHP
```

---

## 10. 常驻内存最佳实践

### 10.1 内存泄漏防范

```php
// ⚠️ 常驻进程最怕内存泄漏

// ❌ 全局变量会一直增长
global $requestLog;
$requestLog[] = $data;  // 内存无限增长！

// ✅ 限制大小
$requestLog = array_slice($requestLog, -1000);

// ❌ 静态变量不会释放
function counter() {
    static $count = 0;
    return ++$count;
}
// 重启进程才会清零

// ✅ 正确处理连接
// 数据库连接要复用，不能每次 new
// Swoole：用协程连接池
// Workerman：在 onWorkerStart 中创建一次

// ✅ max_request 自动重启
$server->set([
    'max_request' => 10000,  // 处理10000请求后重启这个Worker
]);
```

### 10.2 变量隔离

```php
// Swoole/Workerman 都是多进程架构
// 每个进程有独立内存空间

// ⚠️ 全局变量只在当前进程共享！
global $counter;
$counter++;  // 其他进程看不到这个变化

// ✅ 跨进程共享用：
// - Redis（最简单）
// - Swoole Table（内存表）
// - Swoole Atomic（原子计数）

// Swoole Table（高性能跨进程共享内存）
$table = new Swoole\Table(1024);
$table->column('count', Swoole\Table::TYPE_INT);
$table->column('data', Swoole\Table::TYPE_STRING, 256);
$table->create();

$table->set('user_1', ['count' => 0, 'data' => 'hello']);
$table->incr('user_1', 'count', 1);

// Swoole Atomic（无锁原子计数）
$atomic = new Swoole\Atomic(0);
$atomic->add(1);  // 跨进程原子操作
```

### 10.3 热重启

```bash
# Swoole 热重启（不停服）
kill -USR1 <master_pid>   # 重启所有 Worker（旧 Worker 处理完当前请求后退出）
kill -USR2 <master_pid>   # 重启所有 Task Worker

# Workerman 热重启
php http_server.php reload

# 信号含义
# SIGTERM / SIGINT：优雅停止
# SIGUSR1：重载 Worker 进程
```

### 10.4 平滑发布

```php
// 发布策略（零停机）

// 方案1：先启动新服务 → 切换负载均衡 → 停旧服务
// 1. 启动新版服务（新端口）
// 2. Nginx upstream 切换到新端口
// 3. 旧服务优雅停止（kill -TERM）

// 方案2：Swoole 热重载
// kill -USR1 <pid> → 新请求用新代码
// 局限性：不能改进程数等 server->set 中的配置
```

---

## 11. 实战项目

### 11.1 即时通讯（IM）服务

```php
/**
 * 基于 Swoole WebSocket 的简化 IM 服务
 *
 * 功能：登录、单聊、群聊、未读消息
 */
final class IMServer
{
    private array $users = [];       // fd => userId
    private array $userIdToFd = [];  // userId => fd
    private array $rooms = [];       // roomId => [fd1, fd2, ...]

    public function start(): void
    {
        $server = new Swoole\WebSocket\Server('0.0.0.0', 9504);
        $server->set([
            'worker_num'          => 2,
            'enable_coroutine'    => true,
            'heartbeat_check_interval' => 30,
            'heartbeat_idle_time'      => 300,
        ]);

        $server->on('open', [$this, 'onOpen']);
        $server->on('message', [$this, 'onMessage']);
        $server->on('close', [$this, 'onClose']);

        $server->start();
    }

    public function onOpen($server, $request): void
    {
        echo "连接 fd={$request->fd}\n";
    }

    public function onMessage($server, $frame): void
    {
        $data = json_decode($frame->data, true);
        $fd = $frame->fd;

        match ($data['type'] ?? '') {
            'auth' => $this->handleAuth($server, $fd, $data),
            'chat' => $this->handleChat($server, $fd, $data),
            'join_room' => $this->handleJoinRoom($server, $fd, $data),
            'room_chat' => $this->handleRoomChat($server, $fd, $data),
            default => $server->push($fd, json_encode(['type' => 'error', 'msg' => '未知消息类型'])),
        };
    }

    private function handleAuth($server, $fd, array $data): void
    {
        $userId = $data['user_id'];

        // 踢掉旧连接（同一用户多地登录）
        $oldFd = $this->userIdToFd[$userId] ?? null;
        if ($oldFd) {
            $server->push($oldFd, json_encode(['type' => 'kicked', 'msg' => '其他设备登录']));
            $server->close($oldFd);
        }

        $this->users[$fd] = $userId;
        $this->userIdToFd[$userId] = $fd;

        $server->push($fd, json_encode(['type' => 'auth_success', 'user_id' => $userId]));
    }

    private function handleChat($server, $fd, array $data): void
    {
        $fromId = $this->users[$fd] ?? null;
        if (!$fromId) return;

        $toFd = $this->userIdToFd[$data['to']] ?? null;

        $msg = [
            'type'    => 'chat',
            'from'    => $fromId,
            'message' => $data['message'],
            'time'    => date('H:i:s'),
        ];

        if ($toFd) {
            $server->push($toFd, json_encode($msg));
            $server->push($fd, json_encode(['type' => 'sent', 'msg_id' => uniqid()]));
        } else {
            // 离线消息存入 Redis
            $redis = new Redis();
            $redis->connect('127.0.0.1', 6379);
            $redis->lPush("offline:{$data['to']}", json_encode($msg));
            $redis->expire("offline:{$data['to']}", 86400 * 7);
            $server->push($fd, json_encode(['type' => 'offline', 'msg' => '对方不在线，已保存']));
        }
    }

    private function handleJoinRoom($server, $fd, array $data): void
    {
        $roomId = $data['room_id'];
        $this->rooms[$roomId][$fd] = true;
        $server->push($fd, json_encode(['type' => 'joined', 'room_id' => $roomId]));
    }

    private function handleRoomChat($server, $fd, array $data): void
    {
        $roomId = $data['room_id'];
        $fromId = $this->users[$fd] ?? null;

        $msg = [
            'type'    => 'room_chat',
            'room_id' => $roomId,
            'from'    => $fromId,
            'message' => $data['message'],
            'time'    => date('H:i:s'),
        ];

        foreach (($this->rooms[$roomId] ?? []) as $memberFd => $_) {
            if ($memberFd != $fd) {
                $server->push($memberFd, json_encode($msg));
            }
        }
    }

    public function onClose($server, $fd): void
    {
        $userId = $this->users[$fd] ?? null;
        if ($userId) {
            unset($this->userIdToFd[$userId]);
        }
        unset($this->users[$fd]);

        // 清理房间
        foreach ($this->rooms as &$members) {
            unset($members[$fd]);
        }
    }
}

// 启动
(new IMServer())->start();
```

### 11.2 秒杀系统

```php
/**
 * Swoole HTTP 秒杀服务
 *
 * 核心：
 * - Redis 预减库存
 * - 协程并发处理
 * - 队列异步下单
 */
final class SeckillServer
{
    private Redis $redis;
    private Swoole\Coroutine\Channel $orderQueue;

    public function start(): void
    {
        $http = new Swoole\Http\Server('0.0.0.0', 9505);
        $http->set([
            'worker_num'          => 4,
            'enable_coroutine'    => true,
            'max_wait_time'       => 3,
        ]);

        $http->on('WorkerStart', function ($server, $workerId) {
            // 初始化 Redis
            $this->redis = new Redis();
            $this->redis->connect('127.0.0.1', 6379, 0.5);

            // 初始化订单队列
            $this->orderQueue = new Swoole\Coroutine\Channel(10000);

            // 加载秒杀库存到 Redis
            if ($workerId === 0) {
                $this->redis->set('seckill:stock:20260513', 100);
            }

            // 启动订单消费协程
            Swoole\Coroutine::create(function () {
                $this->consumeOrders();
            });
        });

        $http->on('Request', function ($request, $response) {
            $userId = $request->get['user_id'] ?? 0;

            // 1. 频控（同一用户5秒内只允许一次）
            if (!$this->redis->set("seckill:limit:{$userId}", 1, ['nx', 'ex' => 5])) {
                $response->end(json_encode(['code' => -1, 'msg' => '操作太频繁']));
                return;
            }

            // 2. Redis 原子扣减库存
            $stock = $this->redis->decr('seckill:stock:20260513');
            if ($stock < 0) {
                $this->redis->incr('seckill:stock:20260513'); // 回滚
                $response->end(json_encode(['code' => -2, 'msg' => '已售罄']));
                return;
            }

            // 3. 投递到订单队列（异步）
            $this->orderQueue->push([
                'user_id'    => $userId,
                'product_id' => 1001,
                'time'       => microtime(true),
            ], 0.5);

            // 4. 立即返回抢购成功
            $response->end(json_encode([
                'code' => 0,
                'msg'  => '抢购成功，订单处理中',
            ]));
        });

        $http->start();
    }

    private function consumeOrders(): void
    {
        $db = new Swoole\Coroutine\MySQL();
        $db->connect([
            'host'     => '127.0.0.1',
            'user'     => 'root',
            'password' => 'pass',
            'database' => 'mydb',
        ]);

        while (true) {
            $order = $this->orderQueue->pop(5);
            if ($order === false) continue;

            try {
                $db->query("INSERT INTO orders (user_id, product_id, status, created_at)
                            VALUES ({$order['user_id']}, {$order['product_id']}, 'paid', NOW())");

                // 通知用户
                $this->redis->publish('seckill:result', json_encode([
                    'user_id' => $order['user_id'],
                    'status'  => 'paid',
                ]));
            } catch (\Throwable $e) {
                // 失败回滚库存
                $this->redis->incr('seckill:stock:20260513');
                echo "订单创建失败: {$e->getMessage()}\n";
            }
        }
    }
}
```

---

## 12. 性能对比与压测

### 12.1 压测数据参考

```
场景：简单 JSON 接口，4核 8G 服务器

Nginx + PHP-FPM：
  QPS: ~500
  内存: 8 × 30MB = 240MB (8个进程)
  延迟 P99: 200ms

Workerman (4进程)：
  QPS: ~8000
  内存: 常驻 100MB
  延迟 P99: 10ms

Swoole (4Worker + 协程)：
  QPS: ~30000
  内存: 常驻 150MB
  延迟 P99: 5ms

Swoole + 连接池 + 协程：
  QPS: ~50000+
  延迟 P99: 3ms

提升倍数：
  Workerman ≈ 16 倍 PHP-FPM
  Swoole ≈ 60 倍 PHP-FPM
  Swoole协程 ≈ 100 倍 PHP-FPM
```

### 12.2 压测命令

```bash
# ab 压测
ab -n 100000 -c 200 http://localhost:9501/api/status

# wrk（更强）
wrk -t4 -c200 -d30s http://localhost:9501/api/status

# wrk2（恒定吞吐压测）
wrk2 -t4 -c100 -d30s -R10000 http://localhost:9501/api/status
```

---

> 📝 **PHP 不是只能做 Web 请求。掌握进程/线程/协程，配合 Swoole 或 Workerman，PHP 可以胜任高并发、长连接、实时通信等任何服务端场景。**
