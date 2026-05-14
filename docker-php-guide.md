# Docker 部署 PHP 项目实战

> Nginx + PHP-FPM + MySQL + Redis 全家桶一键部署

---

## 目录

1. [Docker 核心概念](#1-docker-核心概念)
2. [安装与环境](#2-安装与环境)
3. [Dockerfile 编写](#3-dockerfile-编写)
4. [docker-compose 编排](#4-docker-compose-编排)
5. [PHP + Nginx 最佳实践](#5-php--nginx-最佳实践)
6. [多阶段构建](#6-多阶段构建)
7. [数据持久化](#7-数据持久化)
8. [网络管理](#8-网络管理)
9. [日志管理](#9-日志管理)
10. [健康检查](#10-健康检查)
11. [CI/CD 集成](#11-cicd-集成)
12. [生产环境最佳实践](#12-生产环境最佳实践)
13. [常用命令速查](#13-常用命令速查)
14. [Docker Swarm 集群](#14-docker-swarm-集群)
15. [Kubernetes 部署 PHP](#15-kubernetes-部署-php)

---

## 1. Docker 核心概念

```
Docker 是什么：
  轻量级容器化平台，将应用和依赖打包在一起运行

核心概念：
  Image（镜像）  → 应用打包的快照（类 = 程序文件）
  Container（容器）→ 镜像的运行实例（对象 = 运行中进程）
  Registry（仓库）→ 镜像存储分发（Docker Hub / 私有 Harbor）
  Volume（卷）    → 持久化数据（不随容器删除丢失）
  Network（网络）  → 容器间通信

Docker vs 虚拟机：
┌──────────────┐  ┌──────────────┐
│   App 1       │  │   App 2       │
├──────────────┤  ├──────────────┤
│   Bin/Libs    │  │   Bin/Libs    │
├──────────────┤  ├──────────────┤
│   Guest OS    │  │   Guest OS    │  ← 虚拟机：每套完整OS
├──────────────┤  ├──────────────┤
│         Hypervisor            │
├───────────────────────────────┤
│         Host OS                │
├───────────────────────────────┤
│         Hardware               │
└───────────────────────────────┘

┌──────────┐ ┌──────────┐ ┌──────────┐
│  App 1    │ │  App 2    │ │  App 3    │
├──────────┤ ├──────────┤ ├──────────┤
│Bin/Libs  │ │Bin/Libs  │ │Bin/Libs  │
├──────────┤ ├──────────┤ ├──────────┤  ← Docker：共享内核
│              Docker Engine         │
├───────────────────────────────────┤
│              Host OS               │
├───────────────────────────────────┤
│              Hardware              │
└───────────────────────────────────┘
```

---

## 2. 安装与环境

```bash
# ═══ 安装 Docker ═══
curl -fsSL https://get.docker.com | bash

# 免 sudo
sudo usermod -aG docker $USER

# Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# ═══ 配置镜像加速（中国） ═══
# /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}

sudo systemctl restart docker
```

---

## 3. Dockerfile 编写

### 3.1 PHP-FPM 镜像

```dockerfile
# Dockerfile
FROM php:8.2-fpm-alpine

LABEL maintainer="dev@example.com"

# ═══ 系统依赖 ═══
RUN apk add --no-cache \
    nginx \
    supervisor \
    libpng-dev \
    libjpeg-turbo-dev \
    freetype-dev \
    libzip-dev \
    libxml2-dev \
    oniguruma-dev \
    curl-dev \
    icu-dev \
    $PHPIZE_DEPS

# ═══ PHP 扩展 ═══
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) \
        pdo_mysql \
        mysqli \
        gd \
        zip \
        bcmath \
        opcache \
        intl \
        soap \
        pcntl \
        sockets

# ═══ Redis 扩展 ═══
RUN pecl install redis \
    && docker-php-ext-enable redis

# ═══ Composer ═══
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# ═══ 优化配置 ═══
RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"
COPY conf/php.ini "$PHP_INI_DIR/conf.d/99-custom.ini"
COPY conf/opcache.ini "$PHP_INI_DIR/conf.d/opcache.ini"
COPY conf/www.conf /usr/local/etc/php-fpm.d/www.conf

# ═══ 工作目录 ═══
WORKDIR /var/www

# ═══ 代码 ═══
COPY . /var/www
RUN chown -R www-data:www-data /var/www

# ═══ Composer 安装依赖 ═══
RUN composer install --no-dev --optimize-autoloader --no-interaction

# ═══ 暴露端口 ═══
EXPOSE 9000

CMD ["php-fpm"]
```

### 3.2 优化的 PHP 配置

```ini
; conf/php.ini
memory_limit = 256M
max_execution_time = 30
max_input_time = 60
post_max_size = 20M
upload_max_filesize = 20M
date.timezone = Asia/Shanghai
```

```ini
; conf/opcache.ini
opcache.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=10000
opcache.revalidate_freq=0           ; 生产环境关掉
opcache.validate_timestamps=0
```

```ini
; conf/www.conf
[www]
user = www-data
group = www-data
listen = 9000
pm = static
pm.max_children = 30
pm.status_path = /status
ping.path = /ping
```

---

## 4. docker-compose 编排

### 4.1 标准 PHP 全家桶

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ═══ Nginx ═══
  nginx:
    image: nginx:alpine
    container_name: app_nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./public:/var/www/public:ro
      - ./storage/app/public:/var/www/storage/app/public:ro
      - ./conf/nginx:/etc/nginx/conf.d:ro
      - ssl_data:/etc/nginx/ssl:ro
    depends_on:
      php:
        condition: service_healthy
    networks:
      - app_net

  # ═══ PHP-FPM ═══
  php:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: app_php
    restart: always
    volumes:
      - .:/var/www:rw
      - composer_cache:/root/.composer:rw
    environment:
      - APP_ENV=production
      - APP_DEBUG=false
    env_file:
      - .env
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "fcgi-client", "9000", "/status"]
      interval: 30s
      timeout: 5s
      retries: 3
    networks:
      - app_net

  # ═══ MySQL ═══
  mysql:
    image: mysql:8.0
    container_name: app_mysql
    restart: always
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_DATABASE}
      MYSQL_USER: ${DB_USERNAME}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./conf/mysql/my.cnf:/etc/mysql/conf.d/custom.cnf:ro
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --default-authentication-plugin=mysql_native_password
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app_net

  # ═══ Redis ═══
  redis:
    image: redis:7-alpine
    container_name: app_redis
    restart: always
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./conf/redis/redis.conf:/usr/local/etc/redis/redis.conf:ro
    command: redis-server /usr/local/etc/redis/redis.conf
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - app_net

volumes:
  mysql_data:
  redis_data:
  composer_cache:
  ssl_data:

networks:
  app_net:
    driver: bridge
```

### 4.2 Nginx 配置（容器版）

```nginx
# conf/nginx/default.conf
server {
    listen 80;
    server_name _;
    root /var/www/public;

    add_header X-Frame-Options "SAMEORIGIN";

    # 日志输出到 stdout（Docker 收集）
    access_log /dev/stdout;
    error_log /dev/stderr;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass php:9000;                    # 容器名解析
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param QUERY_STRING    $query_string;
        fastcgi_param REQUEST_METHOD  $request_method;
        fastcgi_param CONTENT_TYPE    $content_type;
        fastcgi_param CONTENT_LENGTH  $content_length;
        include fastcgi_params;

        fastcgi_read_timeout 60s;
    }

    location ~ /\. {
        deny all;
    }
}
```

---

## 5. PHP + Nginx 最佳实践

### 5.1 多项目共享同一个 Nginx

```yaml
# 一台服务器跑多个 PHP 项目
services:
  nginx_proxy:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
    networks:
      - shared_net

  project_a:
    image: myapp_a:latest
    networks:
      - shared_net

  project_b:
    image: myapp_b:latest
    networks:
      - shared_net

networks:
  shared_net:
```

### 5.2 开发环境热更新

```yaml
# docker-compose.dev.yml
services:
  php:
    volumes:
      - .:/var/www                    # 代码挂载（即时生效）
    environment:
      - APP_ENV=local
      - APP_DEBUG=true

  node:
    image: node:20-alpine
    working_dir: /var/www
    volumes:
      - .:/var/www
    command: npm run dev
```

---

## 6. 多阶段构建

```dockerfile
# ═══ 多阶段构建：编译在 builder 中，最终镜像极简 ═══

# 阶段 1：构建（带编译工具）
FROM php:8.2-fpm-alpine AS builder

RUN apk add --no-cache $PHPIZE_DEPS libpng-dev libzip-dev

RUN docker-php-ext-install pdo_mysql gd zip opcache

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-interaction --no-scripts

COPY . .
RUN composer dump-autoload --optimize

# 删除 dev 依赖和敏感文件
RUN rm -rf tests/ .env.example docker-compose.yml Dockerfile .git


# 阶段 2：运行时（最小镜像）
FROM php:8.2-fpm-alpine

RUN apk add --no-cache libpng libjpeg-turbo libzip icu \
    && docker-php-ext-install -j$(nproc) pdo_mysql gd zip opcache

RUN mv "$PHP_INI_DIR/php.ini-production" "$PHP_INI_DIR/php.ini"

# 从 builder 复制编译好的所有文件
COPY --from=builder /var/www /var/www

RUN chown -R www-data:www-data /var/www

EXPOSE 9000
CMD ["php-fpm"]
```

---

## 7. 数据持久化

```yaml
# ═══ Volume 三种方式 ═══

services:
  app:
    volumes:
      # 1. 命名卷（Docker 管理，推荐）
      - app_data:/var/www/storage

      # 2. 绑定挂载（开发者直接用，性能差一些）
      - ./storage:/var/www/storage

      # 3. tmpfs（内存，重启消失）
      - type: tmpfs
        target: /var/www/cache

volumes:
  app_data:
    driver: local
    driver_opts:
      type: none
      device: /data/app_storage
      o: bind
```

### MySQL 数据备份

```bash
# 备份
docker exec app_mysql mysqldump -u root -p"$PASS" mydb | gzip > backup_$(date +%Y%m%d).sql.gz

# 恢复
gunzip -c backup.sql.gz | docker exec -i app_mysql mysql -u root -p"$PASS" mydb
```

---

## 8. 网络管理

```bash
# ═══ 查看网络 ═══
docker network ls

# ═══ 检查网络详情（看容器IP） ═══
docker network inspect app_net

# ═══ 容器 DNS（容器名自动解析） ═══
# PHP 中连接 MySQL：host => 'mysql'（容器名）
# PHP 中连接 Redis：host => 'redis'（容器名）
```

```yaml
# ═══ 多网络隔离 ═══
services:
  nginx:
    networks:
      - frontend
      - backend

  php:
    networks:
      - backend

  mysql:
    networks:
      - backend
    # Mysql 没有 frontend 网络，外部无法访问

networks:
  frontend:
  backend:
    internal: true   # 禁止访问外网
```

---

## 9. 日志管理

```yaml
# ═══ 统一日志方案（EFK） ═══
services:
  php:
    logging:
      driver: json-file
      options:
        max-size: "100m"
        max-file: "3"
        labels: "app"
        tag: "php-{{.Name}}"

  # Filebeat 收集发送到 Elasticsearch
  filebeat:
    image: docker.elastic.co/beats/filebeat:8.0.0
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

```php
// ═══ 应用日志直接写 stdout（Docker 默认收集） ═══
// PHP-FPM 的 access.log 已经输出到 /dev/stdout
// 自定义日志也可以用 strerr
error_log('订单创建失败: ' . $orderId);
fwrite(STDERR, "任务执行异常\n");
```

---

## 10. 健康检查

```yaml
services:
  php:
    healthcheck:
      test: ["CMD-SHELL", "SCRIPT_NAME=/ping SCRIPT_FILENAME=/ping REQUEST_METHOD=GET cgi-fcgi -bind -connect 127.0.0.1:9000 || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

  mysql:
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${DB_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
```

---

## 11. CI/CD 集成

### 11.1 GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Push to registry
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
          docker tag myapp:${{ github.sha }} registry.example.com/myapp:latest
          docker push registry.example.com/myapp:latest

      - name: Deploy
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: deploy
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/myapp
            docker-compose pull php
            docker-compose up -d --no-deps php
            docker system prune -f
```

### 11.2 零停机部署

```bash
# 滚动更新（docker-compose 不支持，用 docker swarm 或手动）
# 方案：先启动新版容器 → 切换 Nginx upstream → 停旧版

# 1. 新端口启动
docker run -d --name app_v2 -p 9001:9000 app:v2

# 2. Nginx 热重载指向新端口
# upstream backend { server 127.0.0.1:9001; }
docker exec app_nginx nginx -s reload

# 3. 停掉旧版
docker stop app_v1 && docker rm app_v1
```

---

## 12. 生产环境最佳实践

### 12.1 安全检查清单

```dockerfile
# ✅ Dockerfile 安全
FROM php:8.2-fpm-alpine   # 用 alpine 减小攻击面

RUN addgroup -g 1000 app && adduser -u 1000 -G app -D app  # 不用 root
USER app:app

# ❌ 不要
# FROM php:latest          # 版本浮动可能不兼容
# RUN apk add vim git ...  # 不必要的工具
# COPY .env .               # 泄露敏感信息！
```

```yaml
# docker-compose 安全
services:
  php:
    # ❌ 不要 privileged: true
    # ❌ 不要 network_mode: host
    read_only: true           # 只读文件系统
    tmpfs:
      - /tmp
      - /var/www/cache

  mysql:
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}  # 通过 .env 而非硬编码
    # .env 加入 .gitignore
```

### 12.2 资源限制

```yaml
services:
  php:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
```

### 12.3 启动顺序保障

```yaml
# wait-for-it 模式
php:
  depends_on:
    mysql:
      condition: service_healthy  # 等健康检查通过
```

---

## 13. 常用命令速查

```bash
# ═══ 构建 ═══
docker build -t myapp .                          # 当前目录构建
docker build -f Dockerfile.prod -t myapp:prod .  # 指定 Dockerfile

# ═══ Compose ═══
docker-compose up -d                             # 启动所有服务
docker-compose up -d php                         # 启动指定服务
docker-compose down                              # 停止并删除容器
docker-compose restart php                       # 重启
docker-compose logs -f --tail=100 php            # 看日志
docker-compose exec php bash                     # 进容器
docker-compose exec php php artisan migrate      # 执行命令
docker-compose ps                                # 查看状态

# ═══ 镜像 ═══
docker images                                     # 列出镜像
docker rmi myapp:old                              # 删除镜像
docker system prune -a -f                         # 清理无用镜像/容器/卷

# ═══ 容器 ═══
docker ps                                         # 运行中的容器
docker ps -a                                      # 所有容器
docker exec -it app_php sh                        # 进入容器
docker logs -f --tail=50 app_php                  # 容器日志
docker stats                                      # 资源占用
docker inspect app_php                            # 容器详情

# ═══ 调试 ═══
# 看容器内环境变量
docker exec app_php env | sort

# 临时容器调试网络
docker run --rm -it --net app_net alpine sh
# 进去后 ping mysql / nslookup redis

# 导出镜像（离线部署）
docker save myapp:latest | gzip > myapp.tar.gz
# 导入
gunzip -c myapp.tar.gz | docker load
```

> 🐳 **Docker 让环境统一不再是玄学。开发、测试、生产完全一致的运行环境，告别「我机器上没问题」。**

---

## 14. Docker Swarm 集群

### 14.1 Swarm vs Compose vs K8s

```
Docker Compose     → 单机多容器编排（开发/小项目）
Docker Swarm       → 多机集群编排（内置，轻量）
Kubernetes (K8s)   → 大规模集群编排（工业标准）

Swarm 优势：
- Docker 原生内置，无需额外安装
- Compose 文件可直接转 Swarm stack
- 学习成本低，命令简洁
- 适合 3-10 台节点的中小集群
```

### 14.2 初始化集群

```bash
# ═══ 管理节点 ═══
docker swarm init --advertise-addr 192.168.1.100
# 输出 token，其他节点用它加入

# 查看 token
docker swarm join-token worker     # Worker 节点加入命令
docker swarm join-token manager    # Manager 节点加入命令

# ═══ Worker 节点加入 ═══
docker swarm join --token SWMTKN-1-xxx 192.168.1.100:2377

# ═══ 查看集群 ═══
docker node ls
# ID  HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
# 1   node-1    Ready   Active        Leader
# 2   node-2    Ready   Active
# 3   node-3    Ready   Active
```

### 14.3 Stack 部署 PHP 全家桶

```yaml
# docker-stack.yml (基于 Compose 文件改造)
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    configs:
      - source: nginx_conf
        target: /etc/nginx/conf.d/default.conf
    volumes:
      - app_public:/var/www/public:ro
    networks:
      - app_net
    deploy:
      replicas: 2                          # 2 个副本
      update_config:
        parallelism: 1                     # 滚动更新每次 1 个
        delay: 10s
      restart_policy:
        condition: on-failure
        max_attempts: 3

  php:
    image: registry.example.com/myapp:latest
    volumes:
      - app_public:/var/www/public:rw
    environment:
      - APP_ENV=production
    env_file:
      - .env
    networks:
      - app_net
      - backend
    deploy:
      replicas: 3                          # 3 个 PHP Worker
      update_config:
        parallelism: 1
        delay: 15s
      resources:
        limits:
          cpus: '2'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
      healthcheck:
        test: ["CMD", "fcgi-client", "9000", "/ping"]
        interval: 30s
        timeout: 5s
        retries: 3

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_DATABASE}
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - backend
    deploy:
      replicas: 1                          # DB 只能 1 个（单主）
      placement:
        constraints:
          - node.labels.db == true         # 约束到标签节点

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    networks:
      - backend
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == worker

configs:
  nginx_conf:
    file: ./conf/nginx/default.conf

volumes:
  app_public:
    driver: local
  mysql_data:
    driver: local
  redis_data:
    driver: local

networks:
  app_net:
    driver: overlay                       # Swarm overlay 网络
  backend:
    driver: overlay
    internal: true

# ═══ 部署命令 ═══
# docker stack deploy -c docker-stack.yml myapp
# docker stack ls
# docker stack services myapp
# docker service ps myapp_php
# docker service logs -f myapp_php
```

### 14.4 滚动更新

```bash
# ═══ 更新镜像 ═══
docker service update --image registry.example.com/myapp:v2 myapp_php

# ═══ 强制重启 ═══
docker service update --force myapp_php

# ═══ 扩缩容 ═══
docker service scale myapp_php=5          # 扩到 5 个
docker service scale myapp_php=2          # 缩到 2 个

# ═══ 回滚 ═══
docker service rollback myapp_php
```

### 14.5 节点标签与调度

```bash
# ═══ 给节点打标签（实现 DB 绑定/冷热分离） ═══
docker node update --label-add db=true node-1
docker node update --label-add role=worker node-2

# ═══ drain 节点（下线维护） ═══
docker node update --availability drain node-2
# 该节点上的容器会自动迁移到其他节点

# ═══ 恢复节点 ═══
docker node update --availability active node-2
```

### 14.6 Swarm 的局限性

```
Swarm 不适合的场合：
- 节点数 > 100 → 用 K8s
- 需要自动扩缩（HPA）→ K8s 原生支持
- StatefulSet（有序启停、固定网络ID）→ K8s
- 复杂网络策略 → K8s NetworkPolicy
- 多租户/多团队 → K8s Namespace

Swarm 适合的场合：
- 3-10 节点中小规模
- 团队规模小，不想维护 K8s
- 已有 Compose 文件可快速切换
```

---

## 15. Kubernetes 部署 PHP

### 15.1 核心概念

```
K8s 核心对象：

Pod（最小调度单位）
  └─ 一个或多个容器共享网络/IP/存储
  └─ PHP-FPM 是 Pod 内的容器

Deployment（无状态部署）
  └─ 管理 Pod 副本数、滚动更新、回滚

Service（服务发现）
  └─ 给 Pod 提供固定 IP 和 DNS
  └─ ClusterIP / NodePort / LoadBalancer

ConfigMap（配置）
  └─ 环境变量、Nginx 配置等

Secret（密钥）
  └─ DB 密码、API Key（base64 编码）

Ingress（入口）
  └─ HTTP/S 路由到 Service（替代 Nginx 反代层）

PersistentVolumeClaim（持久卷声明）
  └─ 请求存储（类似 Pod 说"我要 10G"）

HPA（水平自动扩缩）
  └─ 根据 CPU/内存自动增减 Pod 数量

Namespace（命名空间）
  └─ 多租户隔离

架构：
  Internet → Ingress → Service → Deployment → Pod(php-fpm)
                                → Deployment → Pod(nginx)
```

### 15.2 完整部署清单

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp

---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: myapp
data:
  APP_ENV: "production"
  APP_DEBUG: "false"
  PHP_MEMORY_LIMIT: "256M"
  OPCAHCE_VALIDATE_TIMESTAMPS: "0"

---
# k8s/nginx-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: myapp
data:
  default.conf: |
    server {
      listen 80;
      root /var/www/public;
      access_log /dev/stdout;
      error_log /dev/stderr;
      location / {
        try_files $uri $uri/ /index.php?$query_string;
      }
      location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
      }
    }

---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: myapp
type: Opaque
stringData:
  DB_PASSWORD: "secret_password"
  REDIS_PASSWORD: ""

---
# k8s/deployment-php.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: php
  template:
    metadata:
      labels:
        app: php
    spec:
      containers:
      - name: php
        image: registry.example.com/myapp:latest
        ports:
        - containerPort: 9000
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secrets
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 2000m
            memory: 512Mi
        livenessProbe:                    # 存活探针
          exec:
            command: ["sh", "-c", "SCRIPT_NAME=/ping SCRIPT_FILENAME=/ping REQUEST_METHOD=GET cgi-fcgi -bind -connect 127.0.0.1:9000"]
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:                   # 就绪探针
          exec:
            command: ["sh", "-c", "SCRIPT_NAME=/ping SCRIPT_FILENAME=/ping REQUEST_METHOD=GET cgi-fcgi -bind -connect 127.0.0.1:9000"]
          initialDelaySeconds: 5
          periodSeconds: 10
        volumeMounts:
        - name: app-code
          mountPath: /var/www
        - name: opcache-tmp
          mountPath: /tmp
      volumes:
      - name: app-code
        persistentVolumeClaim:
          claimName: app-code-pvc
      - name: opcache-tmp
        emptyDir: {}

---
# k8s/deployment-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 500m
            memory: 128Mi
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
        - name: app-code
          mountPath: /var/www/public
          subPath: public
      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config
      - name: app-code
        persistentVolumeClaim:
          claimName: app-code-pvc

---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: myapp
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: myapp
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80

---
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

---
# k8s/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-code-pvc
  namespace: myapp
spec:
  accessModes:
  - ReadWriteMany         # 多 Pod 共享读（Nginx + PHP-FPM）
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs-csi  # NFS / CephFS
```

### 15.3 部署与运维命令

```bash
# ═══ 部署 ═══
kubectl apply -f k8s/                            # 一次性应用所有 YAML
kubectl apply -f k8s/deployment-php.yaml          # 单独更新

# ═══ 查看 ═══
kubectl get pods -n myapp                         # Pod 列表
kubectl get svc -n myapp                          # Service 列表
kubectl get ingress -n myapp                      # Ingress 列表
kubectl get hpa -n myapp                          # 扩缩状态
kubectl describe pod php-xxx -n myapp             # Pod 详情

# ═══ 日志 ═══
kubectl logs -f php-xxx -n myapp                  # 单 Pod 日志
kubectl logs -f -l app=php -n myapp --all-containers  # 所有 PHP Pod 日志

# ═══ 进容器 ═══
kubectl exec -it php-xxx -n myapp -- sh

# ═══ 扩缩 ═══
kubectl scale deployment php --replicas=5 -n myapp

# ═══ 滚动更新 ═══
kubectl set image deployment/php php=registry.example.com/myapp:v2 -n myapp
kubectl rollout status deployment/php -n myapp    # 等更新完成
kubectl rollout undo deployment/php -n myapp      # 回滚
kubectl rollout history deployment/php -n myapp   # 发布历史

# ═══ 调试 ═══
kubectl port-forward php-xxx 9000:9000 -n myapp   # 本地转发调试
kubectl exec -it php-xxx -n myapp -- php artisan migrate  # 执行命令
```

### 15.4 生产环境 Checklist

```yaml
# ═══ 必做 ═══
# ✅ 资源限制（requests/limits）
# ✅ 存活/就绪探针（liveness/readiness）
# ✅ HPA 自动扩缩
# ✅ PodDisruptionBudget（最少可用实例数）
# ✅ 日志集中收集（EFK/Loki）
# ✅ Prometheus 监控
# ✅ NetworkPolicy（网络隔离）
# ✅ Secret 用外部 Vault/SealedSecret

# ═══ 推荐 ═══
# ✅ Helm Chart 管理（模板化）
# ✅ GitOps（ArgoCD/Flux）
# ✅ Service Mesh（Istio）
# ✅ 多可用区部署（反亲和）

# ═══ 反亲和示例（Pod 分散到不同节点） ═══
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: php
        topologyKey: kubernetes.io/hostname
```

> 🐳 **Docker Compose 入门 → Swarm 多机集群 → K8s 工业级编排。根据规模选择，3台 Swarm 够用，30台以上用 K8s。**
