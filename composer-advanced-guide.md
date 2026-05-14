# Composer 进阶实战

> 自动加载原理、私有包、版本约束 — PHP 组件化工程实践

---

## 目录

1. [Composer 核心概念](#1-composer-核心概念)
2. [composer.json 全解](#2-composerjson-全解)
3. [自动加载深入](#3-自动加载深入)
4. [版本约束策略](#4-版本约束策略)
5. [Script 钩子](#5-script-钩子)
6. [私有包发布](#6-私有包发布)
7. [性能优化](#7-性能优化)
8. [常见问题排障](#8-常见问题排障)
9. [最佳实践](#9-最佳实践)
10. [工作流参考](#10-工作流参考)

---

## 1. Composer 核心概念

```
Composer 是什么：
  PHP 的依赖管理工具，类似 npm（Node）、pip（Python）

核心文件：
  composer.json          → 依赖声明（手写/编辑的）
  composer.lock          → 依赖锁定（自动生成，提交到 git）
  vendor/                → 实际安装的包（.gitignore 排除）
  vendor/autoload.php    → 自动加载入口

两个关键操作：
  composer install       → 按 lock 文件精确安装（CI/CD/新环境）
  composer update        → 更新依赖，重写 lock 文件（开发环境）

原则：
  ✅ composer.lock 提交到 git（保证所有人/环境一致）
  ❌ vendor 目录提交到 git
```

---

## 2. composer.json 全解

```json
{
    "name": "mycompany/my-package",

    "description": "一个示例包",
    "type": "library",

    "keywords": ["php", "sdk"],
    "homepage": "https://example.com",
    "license": "MIT",
    "authors": [
        {
            "name": "张三",
            "email": "zhangsan@example.com",
            "role": "Developer"
        }
    ],

    "require": {
        "php": ">=8.1",
        "ext-pdo": "*",
        "ext-redis": "*",
        "monolog/monolog": "^3.0",
        "guzzlehttp/guzzle": "^7.8"
    },

    "require-dev": {
        "phpunit/phpunit": "^10.0",
        "phpstan/phpstan": "^1.10",
        "friendsofphp/php-cs-fixer": "^3.0"
    },

    "suggest": {
        "ext-openssl": "用于加密",
        "symfony/cache": "提供缓存支持"
    },

    "conflict": {
        "php": "<8.0"
    },

    "autoload": {
        "psr-4": {
            "MyCompany\\Package\\": "src/"
        }
    },

    "autoload-dev": {
        "psr-4": {
            "MyCompany\\Package\\Tests\\": "tests/"
        }
    },

    "scripts": {
        "test": "phpunit",
        "check": "phpstan analyse src/ --level=max",
        "cs-fix": "php-cs-fixer fix --allow-risky=yes"
    },

    "config": {
        "optimize-autoloader": true,
        "preferred-install": "dist",
        "sort-packages": true,
        "process-timeout": 300,
        "platform": {
            "php": "8.1.0"
        }
    },

    "minimum-stability": "stable",
    "prefer-stable": true
}
```

---

## 3. 自动加载深入

### 3.1 四种加载方式

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        },
        "psr-0": {
            "OldLib_": "legacy/"
        },
        "classmap": [
            "src/legacy/",
            "database/migrations/"
        ],
        "files": [
            "src/helpers.php",
            "src/constants.php"
        ]
    }
}
```

```
PSR-4（推荐）：
  命名空间前缀 → 目录映射
  "App\\" → "src/"
  App\Controller\UserController → src/Controller/UserController.php

PSR-0（废弃）：
  类名中的下划线 → 目录分隔符
  基本不用了

Classmap（兼容旧代码）：
  扫描目录，生成类名→文件路径的映射
  不遵循 PSR-4 的旧代码用
  composer dump-autoload -o 才生效

Files（全局函数/常量）：
  每次请求都加载这些文件
  用于全局 helper 函数
```

### 3.2 自动加载原理

```php
// vendor/autoload.php 做了什么：

// 1. 加载 Composer 的 ClassLoader
require_once __DIR__ . '/composer/ClassLoader.php';

$loader = new Composer\Autoload\ClassLoader();

// 2. 注册 PSR-4 映射（不扫描，根据规则找文件）
$loader->setPsr4('App\\', __DIR__ . '/../src/');

// 3. 注册 classmap（静态映射，O(1) 查找）
$loader->addClassMap([
    'OldLib_User' => __DIR__ . '/legacy/OldLib/User.php',
]);

// 4. 注册 files（直接 require）
require_once __DIR__ . '/../src/helpers.php';

// 5. 注册到 SPL
$loader->register(true);
```

### 3.3 优化自动加载

```bash
# ═══ 生产环境 ═══
composer dump-autoload -o --no-dev
# -o: 生成 classmap，O(1) 查找（不扫描目录）
# --no-dev: 跳过 dev 依赖

# ═══ 开发环境 ═══
composer dump-autoload
# 不加 -o: PSR-4 动态扫描（新增类无需重新 dump）

# ═══ APCu 缓存 ═══
# composer dump-autoload --apcu
# 把 classmap 存到 APCu，更快（但注意内存）
```

---

## 4. 版本约束策略

```
精确版本:
  1.0.2        → 只安装 1.0.2

范围:
  >=1.0        → 大于等于 1.0
  >=1.0 <2.0   → 1.x 系列
  >=1.0 | <2.0  → 1.0+ 或 <2.0（不太有用）

波浪号（~）:
  ~1.2         → >=1.2.0 <2.0.0   （允许小版本和补丁版本）
  ~1.2.3       → >=1.2.3 <1.3.0   （锁定次版本）

脱字符（^） ← 最常用:
  ^1.2.3       → >=1.2.3 <2.0.0   （不破坏大版本）
  ^0.3.2       → >=0.3.2 <0.4.0   （0.x 版本特殊处理）
  ^2.0         → >=2.0.0 <3.0.0

版本选择规则：
  库/SDK：     ^1.0（宽松，兼容性由包自身保证）
  框架/项目：  ~1.2.3 或精确版本（避免意外）
  composer.lock：提交到 git（确保一致性）
```

```json
{
    "require": {
        // 框架类：较宽松
        "laravel/framework": "^10.0",

        // SDK 类：信任包作者
        "aliyuncs/oss-sdk-php": "^2.4",

        // 关键库：较紧
        "firebase/php-jwt": "~6.10.0",

        // 一直跟着最新
        "phpunit/phpunit": "^10.0",
        "mockery/mockery": "^1.6"
    }
}
```

---

## 5. Script 钩子

```json
{
    "scripts": {
        "post-install-cmd": [
            "@php artisan optimize",
            "@php artisan storage:link"
        ],
        "post-update-cmd": [
            "@php artisan optimize"
        ],
        "post-autoload-dump": [
            "@php artisan package:discover --ansi"
        ],
        "test": [
            "Composer\\Config::disableProcessTimeout",
            "phpunit --colors=always"
        ],
        "lint": "phpstan analyse src/ --level=max",
        "format": "php-cs-fixer fix --allow-risky=yes",
        "check-all": [
            "@lint",
            "@test",
            "@format --dry-run"
        ]
    },

    "scripts-descriptions": {
        "test": "运行单元测试",
        "check-all": "运行全部检查"
    }
}
```

```
可用事件：
  pre-install-cmd        安装前
  post-install-cmd       安装后
  pre-update-cmd         更新前
  post-update-cmd        更新后
  post-autoload-dump     autoload 生成后
  post-root-package-install  根包安装后
  post-create-project-cmd    create-project 后

特殊：
  @php                   用当前环境的 PHP
  @composer              用当前 Composer
  Composer\\Config::disableProcessTimeout  脚本不超时
```

---

## 6. 私有包发布

### 6.1 通过 Git 仓库

```json
{
    "repositories": [
        {
            "type": "vcs",
            "url": "git@github.com:mycompany/private-lib.git"
        },
        {
            "type": "vcs",
            "url": "https://gitlab.internal.com/mycompany/utils.git"
        }
    ],
    "require": {
        "mycompany/private-lib": "^1.0",
        "mycompany/utils": "^2.0"
    }
}
```

```json
// 私有包自身的 composer.json
// github.com/mycompany/private-lib 仓库中
{
    "name": "mycompany/private-lib",
    "description": "内部公共库",
    "type": "library",
    "require": {
        "php": ">=8.1",
        "monolog/monolog": "^3.0"
    },
    "autoload": {
        "psr-4": {
            "MyCompany\\Lib\\": "src/"
        }
    }
}
```

### 6.2 通过 Satis（静态仓库）

```bash
# 1. 创建 satis 项目
composer create-project composer/satis satis

# 2. 配置 satis.json
{
    "name": "mycompany/packages",
    "homepage": "https://packages.mycompany.com",
    "repositories": [
        {"type": "vcs", "url": "git@github.com:mycompany/lib-a.git"},
        {"type": "vcs", "url": "git@github.com:mycompany/lib-b.git"}
    ],
    "require-all": true
}

# 3. 构建
php bin/satis build satis.json public/

# 4. Nginx 指向 public/ 目录
# 5. 项目中引用
composer config repositories.private composer https://packages.mycompany.com
composer require mycompany/lib-a
```

### 6.3 通过 GitLab/Gitea Package Registry

```json
// 发布到 GitLab
{
    "repositories": [
        {
            "type": "composer",
            "url": "https://gitlab.com/api/v4/group/mycompany/-/packages/composer/packages.json"
        }
    ],
    "config": {
        "gitlab-token": {
            "gitlab.com": "glpat-xxxxxxxxxxxx"
        }
    }
}
```

---

## 7. 性能优化

```bash
# ═══ 安装优化 ═══
composer install --no-dev --optimize-autoloader --classmap-authoritative

# --no-dev: 不装 dev 依赖（减少文件数）
# --optimize-autoloader: 生成 classmap
# --classmap-authoritative: 连 PSR-4 扫描都不做（新类不会自动发现）

# ═══ CI/CD ═══
composer install \
    --no-dev \
    --no-interaction \
    --no-progress \
    --no-scripts \
    --optimize-autoloader \
    --classmap-authoritative

# ═══ 镜像加速 ═══
composer config repos.packagist composer https://mirrors.aliyun.com/composer/

# ═══ 并行下载 ═══
composer install --prefer-dist -o

# ═══ 依赖分析 ═══
# 看看谁最占空间
composer show --tree                    # 依赖树
du -sh vendor/*/ | sort -rh | head -10  # 哪个包最大
```

---

## 8. 常见问题排障

```bash
# ═══ 内存不足 ═══
# PHP Fatal error: Allowed memory size exhausted
COMPOSER_MEMORY_LIMIT=-1 composer install
# 或在 composer.json config 中:
# "config": { "process-timeout": 300 }

# ═══ 版本冲突 ═══
# Your requirements could not be resolved
composer why-not vendor/package 2.0
# 看为什么不满足，谁在冲突
composer why vendor/package       # 谁依赖这个包
composer depends vendor/package   # 这个包依赖谁

# ═══ 废弃包检测 ═══
composer outdated -D              # -D 只看直接依赖
composer outdated -m              # 只显示有小版本更新的

# ═══ 安全漏洞 ═══
composer audit                    # 内置安全检查
# 或安装
composer require roave/security-advisories:dev-latest

# ═══ 清除缓存 ═══
composer clear-cache
# 或在报错时，删除 vendor + lock 重来
rm -rf vendor composer.lock && composer install
```

---

## 9. 最佳实践

### 9.1 项目级建议

```bash
# ═══ .gitignore ═══
/vendor/
.env
.phpunit.result.cache
.php-cs-fixer.cache

# composer.lock 提交！（库类包除外）

# ═══ 库（Library）vs 项目（Project） ═══
# 库：composer.lock 不提交到 git
#     别人 require 你，他自己的 lock 文件生效
# 项目：composer.lock 提交到 git
#     所有人 install 得到完全一致的依赖
```

### 9.2 config 建议

```json
{
    "config": {
        "optimize-autoloader": true,     // 默认生产级 autoload
        "preferred-install": "dist",     // 优先 ZIP 不用 git clone
        "sort-packages": true,           // require 按字母排序
        "discard-changes": true          // 避免 dirty vendor
    }
}
```

### 9.3 包命名规范

```
monolog/monolog         ✅ 官方包
mycompany/my-package    ✅ 公司包
hailong/php-utils       ✅  个人包

命名格式：vendor/package-name
- vendor: 组织/用户名
- package-name: 短横线分隔小写英文
- 不要用大写、下划线、斜杠
```

---

## 10. 工作流参考

```bash
# ═══ 新项目 ═══
composer init                          # 交互式创建 composer.json
composer require monolog/monolog       # 安装并加入 require
composer require --dev phpunit/phpunit # 安装 dev 依赖
composer install                       # 别人 clone 后初始化

# ═══ 日常更新 ═══
composer update monolog/monolog        # 更新单个包
composer update                        # 更新全部（慎用！）
composer outdated                      # 看哪些有新版本

# ═══ 代码生成 ═══
composer dump-autoload                 # 重新生成 autoload（不加类时不需要）
composer dump-autoload -o              # 生产优化

# ═══ 全局工具 ═══
composer global require phpstan/phpstan
composer global require friendsofphp/php-cs-fixer
# 放在 ~/.composer/vendor/bin/
# 加到 PATH: export PATH="$HOME/.composer/vendor/bin:$PATH"
```

---

> 📦 **Composer 不只是 install。理解 autoload 原理、版本约束、私有包发布，才是 PHP 组件化工程的完全体。**
