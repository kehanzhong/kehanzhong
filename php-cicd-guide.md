# PHP CI/CD 持续集成与自动化部署实战

> GitHub Actions、Jenkins、Walle 瓦力 — 三种方案从零到生产

---

## 目录

1. [CI/CD 核心概念与管线设计](#1-cicd-核心概念与管线设计)
2. [GitHub Actions — 零成本自动化](#2-github-actions--零成本自动化)
3. [Jenkins — 自建全能 CI 引擎](#3-jenkins--自建全能-ci-引擎)
4. [Walle 瓦力 — 中文友好部署平台](#4-walle-瓦力--中文友好部署平台)
5. [三种方案选型对比](#5-三种方案选型对比)
6. [生产环境最佳实践](#6-生产环境最佳实践)

---

## 1. CI/CD 核心概念与管线设计

### 1.1 基本概念

| 概念 | 说明 |
|------|------|
| **CI** 持续集成 | 代码合入后自动构建、测试 |
| **CD** 持续交付 | 自动构建 + 手动部署 |
| **CD** 持续部署 | 全自动构建 → 测试 → 上线 |
| **Pipeline** 管线 | 从代码提交到部署的完整流程 |
| **Artifact** 产物 | 构建产生的可部署文件（zip/docker 镜像） |

### 1.2 PHP 项目管线设计

```
Push to Git (master/main)
  → 1. Checkout 代码
  → 2. Composer install
  → 3. 代码检查 (phpcs/phpstan)
  → 4. 自动化测试 (PHPUnit)
  → 5. 构建产物 (composer --no-dev + npm build)
  → 6. 安全扫描 (composer audit)
  → 7. 部署到目标服务器
    → 7a. 备份当前版本
    → 7b. 同步文件 (rsync/压缩包)
    → 7c. 执行迁移 (php artisan migrate)
    → 7d. 清理缓存 (opcache:reset)
    → 7e. 健康检查
```

---

## 2. GitHub Actions — 零成本自动化

### 2.1 优势

- **免费**：GitHub 公开仓库无限使用，私有仓库 2000 分钟/月
- **零运维**：不用搭服务器
- **生态丰富**：Marketplace 数千 Action
- **YAML 即文档**：管线定义在 `.github/workflows/` 中，随代码版本化

### 2.2 完整工作流示例

```yaml
# .github/workflows/deploy.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, master]
  pull_request:
    branches: [main, master]

env:
  PHP_VERSION: '8.2'
  COMPOSER_CACHE_DIR: /tmp/composer-cache

jobs:
  # ===== Job 1: 测试 =====
  test:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: test_db
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
          extensions: mbstring, pdo, pdo_mysql, redis, opcache
          coverage: xdebug

      - name: Composer 缓存
        uses: actions/cache@v4
        with:
          path: ${{ env.COMPOSER_CACHE_DIR }}
          key: composer-${{ hashFiles('composer.lock') }}
          restore-keys: composer-

      - name: Install Dependencies
        run: |
          composer install --prefer-dist --no-progress

      - name: 代码规范检查 (PHP CS Fixer)
        run: vendor/bin/php-cs-fixer fix --diff --dry-run

      - name: 静态分析 (PHPStan)
        run: vendor/bin/phpstan analyse --level=5 app/

      - name: 安全扫描
        run: composer audit --no-dev

      - name: 数据库迁移 + 测试
        env:
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: test_db
          DB_USERNAME: root
          DB_PASSWORD: password
        run: |
          php artisan migrate --force
          php artisan test --coverage --min=80

  # ===== Job 2: 部署（仅 push 时）=====
  deploy:
    needs: test
    if: github.event_name == 'push'
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}

      - name: 生产依赖安装
        run: |
          composer install --no-dev --optimize-autoloader --prefer-dist --no-progress
          # 如果有前端
          # npm ci && npm run build

      - name: 打包产物
        run: |
          tar -czf release.tar.gz \
            --exclude='.git' \
            --exclude='storage/logs' \
            --exclude='tests' \
            .

      - name: 上传到服务器
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          source: "release.tar.gz"
          target: "/tmp"

      - name: 远程部署
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /var/www/my-app
            php artisan down --secret="deploy-$(date +%s)"

            tar -xzf /tmp/release.tar.gz -C /tmp
            rsync -a --delete /tmp/my-app/ /var/www/my-app/

            php artisan migrate --force
            php artisan optimize
            php artisan queue:restart

            php artisan up
            echo "✅ Deploy done at $(date)"
```

### 2.3 只跑必要的 Job

```yaml
# 根据变更路径筛选
on:
  push:
    branches: [main]
    paths:
      - 'app/**'
      - 'config/**'
      - 'database/**'
      - 'routes/**'
      - 'composer.json'
      - 'composer.lock'
    paths-ignore:
      - 'docs/**'
      - '*.md'

# 手动触发
on:
  workflow_dispatch:
    inputs:
      environment:
        description: '部署环境'
        required: true
        default: 'staging'
        type: choice
        options: ['staging', 'production']
```

### 2.4 GitHub Actions 缓存策略

```yaml
# Composer 缓存
- uses: actions/cache@v4
  with:
    path: vendor
    key: ${{ runner.os }}-vendor-${{ hashFiles('composer.lock') }}

# node_modules 缓存
- uses: actions/cache@v4
  with:
    path: node_modules
    key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}

# Docker 层缓存
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### 2.5 Secrets 管理

在仓库 Settings → Secrets and variables → Actions 中添加：

```
SERVER_HOST      →  1.2.3.4
SERVER_USER      →  deploy
SERVER_SSH_KEY   →  -----BEGIN OPENSSH PRIVATE KEY----...
DEPLOY_WEBHOOK   →  https://open.feishu.cn/xxx  (通知)
```

---

## 3. Jenkins — 自建全能 CI 引擎

### 3.1 安装（Docker 方式）

```bash
# docker-compose.yml
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts-jdk17
    container_name: jenkins
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock  # 允许 Jenkins 调度 Docker
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
    restart: always

volumes:
  jenkins_home:
```

```bash
docker compose up -d
# 初始密码
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### 3.2 必需插件

安装时选择"推荐插件"，另外手动安装：

| 插件 | 用途 |
|------|------|
| Blue Ocean | 美化 Pipeline UI |
| Git Parameter | 分支/标签 参数选择 |
| Pipeline Utility Steps | Pipeline 工具函数 |
| Docker Pipeline | Pipeline 中使用 Docker |
| Timestamper | 日志时间戳 |
| Build Timestamp | Build 名称加时间戳 |
| Publish Over SSH | SSH 部署到远程服务器 |
| Slack Notification | Slack/飞书通知 |
| Role-based Strategy | 权限管理 |

### 3.3 Pipeline 脚本 (Jenkinsfile)

```groovy
// Jenkinsfile — 放在仓库根目录
pipeline {
    agent any

    environment {
        PHP_IMAGE = 'php:8.2-cli'
        COMPOSER_HOME = "${WORKSPACE}/.composer"
        SERVER_HOST = credentials('server-host')
        SERVER_KEY  = credentials('server-ssh-key')
    }

    parameters {
        choice(
            name: 'DEPLOY_ENV',
            choices: ['staging', 'production'],
            description: '部署环境'
        )
        booleanParam(
            name: 'RUN_MIGRATION',
            defaultValue: true,
            description: '是否执行数据库迁移'
        )
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.BUILD_TAG = "build-${env.BUILD_NUMBER}-${new Date().format('yyyyMMdd-HHmm')}"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    docker run --rm \
                        -v $(pwd):/app \
                        -w /app \
                        ${PHP_IMAGE} \
                        bash -c "
                            apt-get update && apt-get install -y git unzip
                            curl -sS https://getcomposer.org/installer | php
                            php composer.phar install --no-progress --prefer-dist
                        "
                '''
            }
        }

        stage('Code Quality') {
            parallel {
                stage('PHP CS Fixer') {
                    steps {
                        sh 'vendor/bin/php-cs-fixer fix --diff --dry-run || true'
                    }
                }
                stage('PHPStan') {
                    steps {
                        sh 'vendor/bin/phpstan analyse --level=5 app/ || true'
                    }
                }
                stage('Security Audit') {
                    steps {
                        sh 'composer audit --no-dev || true'
                    }
                }
            }
        }

        stage('Test') {
            steps {
                sh '''
                    cp .env.example .env
                    php artisan migrate --force
                    php artisan test --stop-on-failure
                '''
            }
            post {
                always {
                    junit 'tests/reports/junit.xml'  // 测试报告
                }
            }
        }

        stage('Build Artifact') {
            steps {
                sh '''
                    rm -rf .git tests node_modules
                    composer install --no-dev --optimize-autoloader --prefer-dist
                    tar -czf release-${BUILD_TAG}.tar.gz .
                '''
                archiveArtifacts 'release-*.tar.gz'  // 归档
            }
        }

        stage('Deploy') {
            when {
                expression { params.DEPLOY_ENV == 'production' }
                beforeAgent true
            }
            steps {
                script {
                    // 人工审批（生产环境）
                    input message: '确认部署到生产环境？', ok: '部署'
                }
                sh '''
                    scp -i ${SERVER_KEY} release-${BUILD_TAG}.tar.gz deploy@${SERVER_HOST}:/tmp/

                    ssh -i ${SERVER_KEY} deploy@${SERVER_HOST} << 'ENDSSH'
                        cd /var/www/my-app
                        php artisan down

                        tar -xzf /tmp/release-*.tar.gz -C /var/www/my-app
                        php artisan migrate --force
                        php artisan optimize
                        php artisan queue:restart

                        php artisan up
ENDSSH
                '''
            }
        }
    }

    post {
        success {
            // 发送通知
        }
        failure {
            // 发送告警
        }
    }
}
```

### 3.4 多分支自动构建

```groovy
// 在 Jenkins 创建 "Multibranch Pipeline" Job
// 自动扫描仓库所有分支的 Jenkinsfile
// 支持 PR 自动发现
```

### 3.5 共享库复用

```groovy
// vars/phpBuild.groovy — 放在共享库中
def call(Map config = [:]) {
    pipeline {
        agent any
        // 可复用的管线模板...
    }
}

// Jenkinsfile 中使用
@Library('my-shared-library@main') _
phpBuild([
    phpVersion: '8.2',
    deployEnv: 'staging',
])
```

### 3.6 飞书通知集成

```groovy
// 在 post 中：
post {
    failure {
        script {
            def msg = """
                {
                    "msg_type": "interactive",
                    "card": {
                        "header": {
                            "title": {"content": "❌ 构建失败", "tag": "plain_text"},
                            "template": "red"
                        },
                        "elements": [
                            {"tag": "div", "text": {"tag": "plain_text", "content": "项目: ${env.JOB_NAME}"}},
                            {"tag": "div", "text": {"tag": "plain_text", "content": "构建: #${env.BUILD_NUMBER}"}},
                            {"tag": "div", "text": {"tag": "plain_text", "content": "分支: ${env.BRANCH_NAME}"}},
                            {"tag": "action", "actions": [{
                                "tag": "button",
                                "text": {"tag": "plain_text", "content": "查看详情"},
                                "url": "${env.BUILD_URL}"
                            }]}
                        ]
                    }
                }
            """
            sh "curl -X POST -H 'Content-Type: application/json' -d '${msg}' ${FEISHU_WEBHOOK}"
        }
    }
}
```

---

## 4. Walle 瓦力 — 中文友好部署平台

### 4.1 Walle 简介

Walle（瓦力）是国人开发的开源 Web 部署系统，功能类似 Jenkins 但更轻量、配置更友好。支持：

- Web UI 可视化配置
- Git 仓库拉取 + 自动部署
- 多服务器并发部署
- 部署前/后钩子（Shell 脚本）
- 回滚（保留最近 5 个版本）
- 钉钉/企业微信/飞书通知
- 权限管理（RBAC）

**GitHub**: https://github.com/meolu/walle-web

### 4.2 Docker 部署 Walle

```yaml
# docker-compose.yml
version: '3.8'
services:
  walle-web:
    image: alenx/walle-web:2.2
    container_name: walle-web
    ports:
      - "8090:80"
    volumes:
      - /data/walle/logs:/opt/walle-web/logs
      - /data/walle/releases:/opt/walle-web/releases
      - /data/walle/ssh_keys:/home/www-data/.ssh
    environment:
      - WALLE_DB_DSN=tcp(walle-db:3306)/walle
      - WALLE_DB_USER=walle
      - WALLE_DB_PASS=walle_password
    depends_on:
      - walle-db
    restart: always

  walle-db:
    image: mysql:8.0
    container_name: walle-db
    environment:
      - MYSQL_ROOT_PASSWORD=root_password
      - MYSQL_DATABASE=walle
      - MYSQL_USER=walle
      - MYSQL_PASSWORD=walle_password
    volumes:
      - /data/walle/mysql:/var/lib/mysql
    restart: always
```

```bash
docker compose up -d

# 初始化数据库
docker exec walle-web php /opt/walle-web/yii walle/setup

# 默认登录
# 用户名: admin
# 密码: admin
```

### 4.3 Walle 配置流程

#### 第一步：添加项目

后台 → 项目配置 → 创建项目：

```
- 项目名称: My App
- 项目类型: PHP
- 仓库地址: git@github.com:kehanzhong/my-app.git
- 目标集群: 选择目标服务器组
- WebHook 密钥: 自动生成
```

#### 第二步：配置部署服务器

```
- 服务器 IP: 1.2.3.4
- SSH 端口/用户: 22 / deploy
- 服务器上部署路径: /var/www/my-app
- 代码发布目录: /data/releases  (瓦力内部使用)
```

#### 第三步：配置部署流程

**高级任务 → 部署前/后钩子：**

```bash
# === 部署前钩子 ===
# 停止服务
sudo supervisorctl stop laravel-worker

# === 部署后钩子 ===
cd $WWWROOT

# 安装 Composer 依赖
composer install --no-dev --optimize-autoloader --prefer-dist --no-progress

# 安装前端依赖并构建
npm ci && npm run build

# 数据库迁移
php artisan migrate --force

# 优化
php artisan optimize
php artisan view:cache

# 重启队列
sudo supervisorctl restart laravel-worker

# 清理旧版本（保留最近 5 个）
find /data/releases -maxdepth 1 -type d -name 'release_*' | sort -r | tail -n +6 | xargs rm -rf

# 健康检查
curl -f http://localhost/health || exit 1

echo "✅ 部署完成 $(date)"
```

#### 第四步：Git WebHook 自动触发

在 GitHub/GitLab 仓库 Settings → Webhooks 中添加：

```
Payload URL:  https://walle.example.com/walle/hook?token=YOUR_HOOK_TOKEN
Content type: application/json
Events:       Just the push event
```

配置后，每次 `git push` 到指定分支会自动触发部署。

### 4.4 回滚操作

瓦力自动保留最近 5 个发布版本。回滚步骤：

1. Walle UI → 项目 → 上线单历史
2. 选择要回滚的版本
3. 点击"回滚"按钮

自动执行：将 `current` 软链接切换到指定版本目录。

### 4.5 多服务器并发部署

在"目标集群"中添加多台服务器，瓦力会自动：

1. 逐台或并行拉取代码
2. 在每台服务器执行相同的前/后钩子
3. 单台失败则整体标记失败
4. 支持分批部署（先灰度 1 台，验证后再全量）

### 4.6 瓦力 vs Jenkins 对比

| 维度 | Walle 瓦力 | Jenkins |
|------|-----------|---------|
| 安装难度 | 简单（Docker 一条龙） | 中等 |
| 中文支持 | ✅ 原生中文 | 需插件 |
| PHP 专属 | ✅ 深度优化 | 通用 |
| 扩展性 | 钩子脚本 | ⭐ 插件生态顶级 |
| 回滚 | ✅ 一键回滚 | 需自行实现 |
| 多服务器 | ✅ 天然支持 | 需配置 |
| 构建能力 | 较弱（专注部署） | ⭐ 极其强大 |
| 学习曲线 | 低 | 高 |

---

## 5. 三种方案选型对比

| | GitHub Actions | Jenkins | Walle 瓦力 |
|---|---|---|---|
| **托管** | 云（GitHub） | 自建 | 自建 |
| **成本** | 免费（2000分钟/月） | 服务器成本 | 服务器成本 |
| **上手难度** | ⭐ 低 | ⭐⭐⭐ 高 | ⭐⭐ 中 |
| **CI 能力** | ⭐⭐⭐ 强 | ⭐⭐⭐ 强 | ⭐ 弱 |
| **CD 能力** | ⭐⭐ 中 | ⭐⭐⭐ 强 | ⭐⭐⭐ 强 |
| **回滚** | 自行实现 | 自行实现 | ✅ 原生支持 |
| **多服务器** | 需自行编排 | 需自行编排 | ✅ 原生支持 |
| **通知** | Slack/Discord | 插件丰富 | 飞书/钉钉/企微 |
| **中文生态** | 无 | 弱 | ✅ 强 |
| **适合团队** | 1-50 人 | 50+ 人 | 5-30 人（国内） |

### 推荐组合

```
个人/小团队 (1-10人):
  GitHub Actions (CI + 测试) → 瓦力 (部署 + 回滚)

中大型团队:
  GitHub Actions / Jenkins (CI) → 瓦力 (部署) / 自研 CD 平台

纯内网环境:
  Gitea + Jenkins + 瓦力
```

---

## 6. 生产环境最佳实践

### 6.1 部署策略

```bash
# 零停机部署 (Laravel)
php artisan down --secret="secret-key"  # 维护模式 + 秘钥绕过
# ... 部署操作 ...
php artisan up

# 软链接切换（原子操作）
mkdir -p /data/releases/release_$(date +%Y%m%d%H%M%S)
ln -sfT /data/releases/release_20240501120000 /var/www/current
```

### 6.2 健康检查

```php
// routes/web.php
Route::get('/health', function () {
    $checks = [
        'database' => checkDatabase(),
        'redis'    => checkRedis(),
        'storage'  => is_writable(storage_path()),
    ];

    $allPassed = !in_array(false, $checks);

    return response()->json([
        'status'  => $allPassed ? 'healthy' : 'degraded',
        'checks'  => $checks,
        'version' => config('app.version'),
        'time'    => now()->toIso8601String(),
    ], $allPassed ? 200 : 503);
});

function checkDatabase(): bool {
    try {
        DB::connection()->getPdo();
        return true;
    } catch (\Exception $e) {
        return false;
    }
}

function checkRedis(): bool {
    try {
        Redis::ping();
        return true;
    } catch (\Exception $e) {
        return false;
    }
}
```

### 6.3 通知链路

```
构建失败 → 飞书/钉钉/Slack 通知 → @相关人
部署成功 → 通知 + 变更日志摘要
健康检查失败 → 立即告警（短信/电话）
```

### 6.4 安全清单

```yaml
# ✅ 凭证不硬编码 — 用 Secrets / 环境变量
# ✅ SSH Key 专用 — 为 CI 创建专用 deploy 用户
# ✅ 最小权限 — deploy 用户只能操作 /var/www/my-app
# ✅ 网络隔离 — 部署服务器仅允许 CI 服务器 IP 访问 22 端口
# ✅ 审计日志 — 所有部署记录保留 90 天
# ✅ 回滚验证 — 每个版本保留 5 个历史版本
# ✅ 灰度发布 — 先部署到 1 台，验证后再全量
```

### 6.5 完整部署脚本（无服务中断）

```bash
#!/bin/bash
set -e

TIMESTAMP=$(date +%Y%m%d-%H%M%S)
RELEASE_DIR="/data/releases/release_${TIMESTAMP}"
APP_DIR="/var/www/my-app"
KEEP_RELEASES=5

echo "🚀 开始部署 — ${TIMESTAMP}"

# 1. 拉取代码
git clone --depth=1 git@github.com:kehanzhong/my-app.git "${RELEASE_DIR}"
cd "${RELEASE_DIR}"

# 2. 安装依赖
composer install --no-dev --optimize-autoloader --prefer-dist --no-progress
npm ci && npm run build

# 3. 同步持久化文件
for DIR in storage uploads .env; do
    if [ -e "${APP_DIR}/shared/${DIR}" ]; then
        ln -sfT "${APP_DIR}/shared/${DIR}" "${RELEASE_DIR}/${DIR}"
    fi
done

# 4. 数据库迁移
php artisan migrate --force

# 5. 原子切换
ln -sfT "${RELEASE_DIR}" "${APP_DIR}"

# 6. 重启服务
sudo supervisorctl restart laravel-worker:*

# 7. OPcache 重置
curl -s http://localhost/opcache-reset || true

# 8. 清理旧版本
ls -dt /data/releases/release_* | tail -n +$((KEEP_RELEASES + 1)) | xargs rm -rf

echo "✅ 部署完成 — ${TIMESTAMP}"
```

---

## 附录 A: 常用命令速查

```bash
# GitHub Actions 本地调试
brew install act                # macOS
act push                        # 模拟 push 事件
act -j test                     # 只跑 test Job

# Jenkins CLI
java -jar jenkins-cli.jar -s http://localhost:8080 list-jobs
java -jar jenkins-cli.jar -s http://localhost:8080 build my-job

# Docker CI 调试
docker run --rm -v $(pwd):/app -w /app php:8.2-cli vendor/bin/phpunit

# 检查 PHP 扩展
php -m | grep -E "mbstring|pdo|redis|opcache"
```

## 附录 B: 常见问题

**Q: GitHub Actions 如何访问内网服务器？**
A: 使用 GitHub Self-hosted Runner，将 Runner 部署在内网服务器上。

**Q: Walle 和 Jenkins 能一起用吗？**
A: 可以。Jenkins 负责 CI（构建/测试），Walle 负责 CD（多服务器部署 + 回滚）。

**Q: 部署时 502 怎么办？**
A: PHP-FPM 在 `php artisan optimize` 后需重载：`sudo systemctl reload php8.2-fpm`

---

> **文档版本**: 1.0  
> **更新日期**: 2026-05-14  
> **适用场景**: PHP 项目 CI/CD 生产级自动化部署
