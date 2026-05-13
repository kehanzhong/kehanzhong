# Nginx + PHP-FPM 配置实战

> 从零到生产，location/rewrite/HTTPS/负载均衡/性能调优

---

## 目录

1. [核心概念](#1-核心概念)
2. [安装与基础配置](#2-安装与基础配置)
3. [location 匹配规则](#3-location-匹配规则)
4. [rewrite 详解](#4-rewrite-详解)
5. [静态资源优化](#5-静态资源优化)
6. [HTTPS 配置](#6-https-配置)
7. [负载均衡](#7-负载均衡)
8. [WebSocket 反代](#8-websocket-反代)
9. [限流与安全](#9-限流与安全)
10. [PHP-FPM 调优](#10-php-fpm-调优)
11. [Nginx 调优参数](#11-nginx-调优参数)
12. [日志与监控](#12-日志与监控)
13. [常见场景实战](#13-常见场景实战)
14. [排障指南](#14-排障指南)

---

## 1. 核心概念

```
Nginx 是什么：
  一个异步非阻塞的 Web 服务器 / 反向代理服务器
  事件驱动（epoll），单 master + 多 worker 架构

请求流向：
  浏览器 → Nginx(:80) → PHP-FPM(:9000) → PHP 代码 → MySQL/Redis

Nginx 多进程模型：
  ┌─────────────┐
  │ Master 进程  │  ← 读取配置、管理 Worker
  └──────┬──────┘
    ┌────┼────┬────────┐
    ▼    ▼    ▼        ▼
  Worker Worker Worker Worker  ← 处理请求（共享监听端口）
  
  每个 Worker 是单线程、非阻塞
  一个 Worker 可以处理数千并发连接
```

---

## 2. 安装与基础配置

```bash
# ═══ 安装 ═══
# Ubuntu/Debian
apt-get install -y nginx

# CentOS/RHEL
yum install -y nginx

# 启动
systemctl start nginx
systemctl enable nginx

# ═══ php-fpm ═══
apt-get install -y php8.2-fpm

# 启动
systemctl start php8.2-fpm
```

```nginx
# /etc/nginx/nginx.conf — 主配置骨架
user  www-data;
worker_processes  auto;          # CPU 核数
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  10240;   # 每个 Worker 最大连接数
    use  epoll;                  # Linux 使用 epoll
    multi_accept  on;            # 一次接受多个连接
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # 日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" '
                      'rt=$request_time';

    access_log  /var/log/nginx/access.log  main;

    sendfile    on;              # 零拷贝传输
    tcp_nopush  on;              # 攒包发送
    tcp_nodelay on;              # keep-alive 下不发小包

    keepalive_timeout  65;
    client_max_body_size  20m;   # 上传限制
    client_body_buffer_size 128k;

    # Gzip 压缩
    gzip  on;
    gzip_min_length  1k;
    gzip_comp_level  2;          # 1-9，越大越慢
    gzip_types  text/plain text/css text/javascript
                application/json application/javascript
                application/xml application/xml+rss;
    gzip_vary  on;
    gzip_proxied  any;

    # 虚拟主机
    include /etc/nginx/conf.d/*.conf;
}
```

---

## 3. location 匹配规则

### 3.1 匹配语法

```
location [ = | ~ | ~* | ^~ ] /uri/ { ... }

优先级（从高到低）：
1. =   精确匹配          = /api/status
2. ^~  前缀匹配（不再检查正则）  ^~ /static/
3. ~   区分大小写正则    ~ \.php$
4. ~*  不区分大小写正则  ~* \.(jpg|png|gif)$
5. 无修饰符 前缀匹配        /api/

同级匹配：最长前缀优先，顺序无关
正则匹配：按配置顺序，第一个命中即停
```

```nginx
server {
    listen 80;
    server_name example.com;

    # 优先级 1: 精确匹配
    location = /favicon.ico {
        return 204;
    }

    # 优先级 2: 前缀匹配（不检查正则）
    location ^~ /static/ {
        alias /var/www/public/static/;
        expires 30d;
    }

    # 优先级 3/4: 正则匹配
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    # 优先级 5: 前缀匹配（兜底）
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
}
```

### 3.2 alias vs root

```nginx
# root：拼接 root + location
# alias：用 alias 替换 location

# 请求 /static/js/app.js

location /static/ {
    root /var/www/html;
    # → /var/www/html/static/js/app.js ✅
}

location /download/ {
    alias /data/files/;
    # → /data/files/js/app.js ✅（/download 被替换）
}

# ⚠️ alias 必须以 / 结尾！
# ⚠️ alias 不能用在正则 location
```

---

## 4. rewrite 详解

### 4.1 基础用法

```nginx
# rewrite regex replacement [flag]

# flag 说明：
# last    → 重写后重新走 location 匹配
# break   → 重写后不再匹配 location（在当前上下文中继续）
# redirect → 302 临时重定向
# permanent → 301 永久重定向

# ═══ 常见用法 ═══

# URL 美化（去掉 index.php）
location / {
    try_files $uri $uri/ /index.php?$query_string;
}

# 强制跳转 HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$server_name$request_uri;
}

# 301 重定向（不带 www → 带 www）
server {
    listen 80;
    server_name example.com;
    return 301 http://www.example.com$request_uri;
}

# 旧 URL 迁移
rewrite ^/old-products/(.*)$ /products/$1 permanent;

# API 版本转发
rewrite ^/api/v1/(.*)$ /api/v2/$1 last;

# 路径不存在时兜底
location / {
    if (!-e $request_filename) {
        rewrite ^/(.*)$ /index.php?/$1 last;
    }
}
```

### 4.2 try_files 最佳实践

```nginx
# Laravel / Symfony（入口文件模式）
location / {
    try_files $uri $uri/ /index.php?$query_string;
}

# ThinkPHP
location / {
    if (!-e $request_filename) {
        rewrite ^/(.*)$ /index.php?s=/$1 last;
    }
}

# Yii2
location / {
    try_files $uri $uri/ /index.php$is_args$args;
}

# SPA（Vue/React）
location / {
    try_files $uri $uri/ /index.html;
}
# ⚠️ SPA 的 API 路由要放前面
location /api/ {
    try_files $uri @api_fallback;
}
```

---

## 5. 静态资源优化

```nginx
# ═══ 完整静态资源方案 ═══
location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
    expires 30d;                          # 强缓存 30 天
    add_header Cache-Control "public, immutable";
    access_log off;                       # 不记日志
    gzip_static on;                       # 优先用已压缩的 .gz 文件
    try_files $uri =404;
}

# ═══ 预压缩 ═══
# 提前生成 gzip 文件，Nginx 直接发送（零CPU）
# find /static -type f -name "*.css" -o -name "*.js" | xargs -I {} gzip -k9 {}

# ═══ 图片懒加载配合 ═══
# 1x1 占位图
location = /placeholder.gif {
    empty_gif;
}
```

---

## 6. HTTPS 配置

```nginx
# ═══ 完整 HTTPS 配置 ═══
server {
    listen 443 ssl http2;
    server_name example.com;

    # 证书（Let's Encrypt）
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # 安全协议
    ssl_protocols TLSv1.2 TLSv1.3;            # 禁用 TLSv1.0/1.1
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;

    # HSTS（强制 HTTPS，1年）
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # 安全头
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # OCSP 装订（加速）
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    # Session 缓存（减少握手）
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    root /var/www/public;
    # ... location 配置 ...
}

# HTTP → HTTPS 强制跳转
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

# ═══ Let's Encrypt 自动签发 ═══
# certbot --nginx -d example.com -d www.example.com
```

---

## 7. 负载均衡

### 7.1 upstream 调度算法

```nginx
# ═══ 基本 LB ═══
upstream backend {
    # 调度算法（默认轮询）
    # least_conn;  最少连接
    # ip_hash;     IP 哈希（会话保持）
    # hash $request_uri;  URL 哈希

    server 192.168.1.10:9000 weight=3 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:9000 weight=1;
    server 192.168.1.12:9000 backup;    # 备用（其他全挂才用）
    # server 192.168.1.13:9000 down;    # 永久下线

    keepalive 32;   # 到后端的 keep-alive 连接数
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 超时
        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;

        # 缓冲
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 16k;
    }
}
```

### 7.2 健康检查

```nginx
# Nginx 商业版有 health_check，开源版用 max_fails
upstream backend {
    server 192.168.1.10:9000 max_fails=3 fail_timeout=30s;
    server 192.168.1.11:9000 max_fails=3 fail_timeout=30s;
    # max_fails=3, fail_timeout=30s：
    #   30秒内失败3次 → 标记为 down → 30秒后重试
}

# 主动健康检查（Nginx Plus 或 nginx_upstream_check_module）
# check interval=3000 rise=2 fall=3 timeout=1000 type=http;
# check_http_send "HEAD /health HTTP/1.0\r\n\r\n";
# check_http_expect_alive http_2xx http_3xx;
```

### 7.3 多站点负载均衡

```nginx
# 同一 Nginx 对多个后端站点做 LB
upstream site_a {
    server 10.0.0.1:80;
    server 10.0.0.2:80;
}
upstream site_b {
    server 10.0.0.3:80;
    server 10.0.0.4:80;
}

server {
    listen 80;
    server_name a.example.com;
    location / {
        proxy_pass http://site_a;
    }
}
server {
    listen 80;
    server_name b.example.com;
    location / {
        proxy_pass http://site_b;
    }
}
```

---

## 8. WebSocket 反代

```nginx
server {
    listen 443 ssl http2;
    server_name ws.example.com;

    location /ws/ {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;

        # ⚠️ WebSocket 必须设置这两行
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

        # 长连接超时（比 HTTP 长得多）
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}

upstream ws_backend {
    server 127.0.0.1:9503;  # Swoole WebSocket
    server 127.0.0.1:9504;
}
```

---

## 9. 限流与安全

### 9.1 请求频率限制

```nginx
# ═══ 定义限流区 ═══
# 每个 IP 每秒最多 10 个请求
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

# 连接数限制
limit_conn_zone $binary_remote_addr zone=addr:10m;

server {
    # ═══ 请求速率限制 ═══
    location /api/ {
        # burst=20: 突发允许排队 20 个
        # nodelay: 不延迟处理 burst 内的请求
        limit_req zone=mylimit burst=20 nodelay;
        limit_req_status 429;

        proxy_pass http://backend;
    }

    # ═══ 下载限速 ═══
    location /download/ {
        limit_rate 500k;           # 每连接 500KB/s
        limit_rate_after 10m;      # 前 10MB 不限速
    }

    # ═══ 单IP连接数限制 ═══
    location / {
        limit_conn addr 10;
        limit_conn_status 503;
    }

    # ═══ 并发限流（防止高并发击穿） ═══
    location /seckill/ {
        limit_conn addr 1;         # 同IP只能1个连接
        limit_req zone=mylimit burst=5;
        proxy_pass http://backend;
    }
}
```

### 9.2 安全加固

```nginx
# ═══ 完整安全配置 ═══
server {
    # 隐藏版本号（防止利用已知漏洞）
    server_tokens off;

    # 禁止通过 IP 访问
    # 默认 server 块
    # server { listen 80 default_server; return 444; }

    # 只允许指定方法
    if ($request_method !~ ^(GET|HEAD|POST|PUT|DELETE)$) {
        return 405;
    }

    # 防止直接访问敏感文件
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # 防盗链
    location ~* \.(jpg|png|gif)$ {
        valid_referers none blocked example.com *.example.com;
        if ($invalid_referer) {
            return 403;
        }
    }

    # 限制 User-Agent（防扫描）
    if ($http_user_agent ~* (nmap|nikto|wikto|sf|sqlmap|bsqlbf|w3af|acunetix|havij|appscan)) {
        return 403;
    }

    # IP 黑名单
    deny 58.218.0.0/16;   # 徐州某段
    allow all;
}
```

---

## 10. PHP-FPM 调优

### 10.1 进程管理模式

```ini
; /etc/php/8.2/fpm/pool.d/www.conf

; ═══ 进程管理模式 ═══
pm = dynamic
; static：固定进程数（内存够用首选）
; dynamic：按需增减（小站点省内存）
; ondemand：完全按需（不适合生产，延迟高）

; 需要提前评估：
;   最大进程数 = 可用内存 / 每个进程的内存占用
;   例：8G 内存，200MB/进程 → 最多 40 个进程
;   留 2G 给 MySQL/Redis/系统 → 实际 30 个

pm.max_children = 30              ; 最大子进程数
pm.start_servers = 10             ; 启动时进程数
pm.min_spare_servers = 5          ; 最小空闲进程
pm.max_spare_servers = 20         ; 最大空闲进程

; ═══ static 模式（推荐生产） ═══
; pm = static
; pm.max_children = 30

; ═══ 请求限制 ═══
pm.max_requests = 10000           ; 处理 1 万请求后回收（防内存泄漏）
request_terminate_timeout = 30s   ; 单个请求最大 30 秒

; ═══ 慢日志 ═══
slowlog = /var/log/php-fpm-slow.log
request_slowlog_timeout = 5s      ; 超过 5 秒记录

; ═══ 状态监控 ═══
pm.status_path = /status
ping.path = /ping
```

### 10.2 Nginx 配合 FPM status

```nginx
# 开启 FPM 状态页面
location = /status {
    fastcgi_pass unix:/run/php/php8.2-fpm.sock;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
    # IP 白名单
    allow 127.0.0.1;
    allow 10.0.0.0/8;
    deny all;
}

# 访问 /status 返回：
# pool:                 www
# process manager:      dynamic
# start time:           13/May/2026:10:00:00 +0800
# accepted conn:        150000
# listen queue:         0
# max listen queue:     10
# idle processes:       5
# active processes:     10
# total processes:      15
# max active processes: 20
# max children reached: 0
# slow requests:        2
```

### 10.3 PHP-FPM 性能调优

```ini
; 关键 php.ini 调优
memory_limit = 256M             ; 不要太大（多进程累加）
max_execution_time = 30         ; 别超过 request_terminate_timeout
opcache.enable = 1
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 10000
opcache.validate_timestamps = 1 ; 开发开，生产建议0
opcache.revalidate_freq = 2

; ⚠️ 生产环境建议关掉时间戳检查
; opcache.validate_timestamps = 0  → 代码改了要 reload fpm
```

---

## 11. Nginx 调优参数

```nginx
# ═══ 核心调优 ═══
worker_processes  auto;
worker_rlimit_nofile  65535;      # 每个 Worker 最大文件描述符

events {
    worker_connections  10240;
    use  epoll;
    multi_accept  on;
}

http {
    sendfile  on;
    tcp_nopush  on;
    tcp_nodelay  on;

    keepalive_timeout  65;
    keepalive_requests 100;

    # 上游 keep-alive
    upstream backend {
        server 127.0.0.1:9000;
        keepalive 32;             # 到后端的 keep-alive 连接池
    }

    # 代理缓冲
    proxy_buffer_size 4k;
    proxy_buffers 8 16k;
    proxy_busy_buffers_size 32k;
    proxy_temp_file_write_size 256k;

    # 客户端缓冲
    client_body_buffer_size 128k;
    client_max_body_size 20m;

    # 隐藏版本
    server_tokens off;
}
```

---

## 12. 日志与监控

### 12.1 日志切割

```bash
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily           # 每天轮转
    missingok
    rotate 30       # 保留 30 天
    compress
    delaycompress
    notifempty
    dateext
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

### 12.2 自定义日志

```nginx
# 记录请求时间超过 1 秒的请求
map $request_time $loggable {
    ~^0\.[0-9]+ 0;   # < 1s 不记录
    default     1;    # >= 1s 记录
}

access_log /var/log/nginx/access.log main if=$loggable;

# 错误日志级别：debug / info / notice / warn / error / crit
error_log /var/log/nginx/error.log warn;
```

---

## 13. 常见场景实战

### 13.1 动静分离

```nginx
server {
    listen 80;
    server_name www.example.com;
    root /var/www/public;

    # 静态资源：Nginx 直接返回
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff2)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # 动态请求：转发 PHP-FPM
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

### 13.2 CDN 代理

```nginx
# CDN 回源取真实 IP
set_real_ip_from 103.21.244.0/22;     # CloudFlare
set_real_ip_from 173.245.48.0/20;
real_ip_header X-Forwarded-For;

# 或者通用方案
real_ip_header X-Forwarded-For;
real_ip_recursive on;
set_real_ip_from 0.0.0.0/0;           # 信任所有代理（仅内网）
```

### 13.3 跨域 CORS

```nginx
location /api/ {
    # 允许跨域
    add_header Access-Control-Allow-Origin $http_origin;
    add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, OPTIONS";
    add_header Access-Control-Allow-Headers "Authorization, Content-Type";
    add_header Access-Control-Allow-Credentials "true";

    # 预检请求直接返回
    if ($request_method = OPTIONS) {
        return 204;
    }

    proxy_pass http://backend;
}
```

### 13.4 灰度发布

```nginx
# 通过 Cookie 控制灰度
split_clients "${remote_addr}AAA" $variant {
    10% "canary";
    90% "stable";
}

upstream canary {
    server 127.0.0.1:8081;  # 灰度版本
}
upstream stable {
    server 127.0.0.1:8080;  # 稳定版本
}

map $variant $backend {
    "canary" "canary";
    "stable" "stable";
}

server {
    location / {
        proxy_pass http://$backend;
    }
}
```

### 13.5 图片动态裁切

```nginx
# Nginx image_filter 模块（需编译 --with-http_image_filter_module）
location ~ ^/img/(\d+)x(\d+)/(.*)$ {
    set $w $1;
    set $h $2;
    set $img $3;

    image_filter resize $w $h;
    image_filter_jpeg_quality 80;
    image_filter_buffer 10M;
    alias /var/www/storage/$img;
}
```

---

## 14. 排障指南

```bash
# ═══ 配置检查 ═══
nginx -t                     # 测试配置语法
nginx -T                     # 测试并打印完整配置

# ═══ 运行状态 ═══
nginx -s reload              # 热重载（不停机）
nginx -s stop                # 快速停止
nginx -s quit                # 优雅停止（处理完当前请求）
nginx -s reopen              # 重新打开日志文件

# ═══ 常见问题 ═══

# 1. 502 Bad Gateway
# PHP-FPM 没启动 / 进程不够 / 超时
# 查: systemctl status php8.2-fpm
# 查: tail -f /var/log/php8.2-fpm.log
# 解决: 调大 pm.max_children / request_terminate_timeout

# 2. 504 Gateway Timeout
# PHP 执行时间超过 proxy_read_timeout
# 解决: 增大 fastcgi_read_timeout / proxy_read_timeout

# 3. 413 Request Entity Too Large
# 上传文件超过 client_max_body_size
# 解决: client_max_body_size 100m;

# 4. 404 但文件存在
# 检查 root 路径 / try_files / location 匹配顺序
# nginx -T | grep -A5 "location"

# 5. Permission denied
# nginx worker 用户没权限读文件 / 或 SELinux
# 解决: chmod 755 / chown www-data / setenforce 0

# 6. listen queue overflow
# 请求太多，Worker 来不及 accept
# 解决: 增大 net.core.somaxconn
# sysctl -w net.core.somaxconn=65535
```

---

> 💡 **Nginx = 高效反向代理 + 静态服务器。配合 PHP-FPM 的 process manager 和 OpCache，可轻松支撑日千万级 PV。**
