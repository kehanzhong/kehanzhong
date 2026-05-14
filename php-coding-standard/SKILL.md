---
name: php-coding-standard
description: PHP 开发规范与最佳实践。使用 PSR-12 编码标准，包含代码质量检查、规范参考和自动化工具。当需要：(1) 创建或修改 PHP 代码时，(2) 检查代码是否符合规范时，(3) 设置项目编码标准时，使用本技能。
---

# PHP 开发规范

## 概述

本技能提供完整的 PHP 开发规范参考，基于 PSR-12 标准，包含代码质量检查工具和最佳实践。

## 当使用本技能

✅ **使用本技能当：**

- 创建新的 PHP 项目
- 修改现有 PHP 代码以符合规范
- 设置 CI/CD 中的代码检查
- 教学或参考 PHP 最佳实践

❌ **不使用本技能当：**

- 仅需简单语法帮助 → 查阅 PHP 官方文档
- 修复具体 bug → 直接调试代码
- 性能优化 → 使用性能分析工具

## 核心规范

### 1. 基础规范 (PSR-1)

- 文件必须以 `<?php` 或 `<?=` 开头
- 类名使用 StudlyCaps 驼峰命名
- 方法名使用 camelCase 驼峰命名
- 常量名使用全大写下划线分隔

### 2. 代码样式 (PSR-12)

- 使用 4 个空格缩进，不使用 Tab
- 每行不超过 120 个字符
- 使用 Unix LF 换行符
- 文件末尾保留一个空行

### 3. 命名规范

**类名：**
```php
class UserController extends BaseController
{
    // ...
}
```

**方法名：**
```php
public function getUserById($id)
{
    // ...
}
```

**变量名：**
```php
$userId = 123;
$userName = "John";
```

**常量：**
```php
const MAX_LENGTH = 100;
```

## 使用方法

### 查看规范参考

参考 `references/` 目录下的文档：

- `psr-12-coding-style-guide.md` - 完整 PSR-12 标准
- `best-practices.md` - PHP 最佳实践

### 运行代码检查

```bash
# 检查单个文件
php-cs-fixer fix /path/to/file.php --diff

# 检查整个项目
php-cs-fixer fix /path/to/project --diff

# 使用自定义规则
php-cs-fixer fix /path/to/project --config=.php-cs-fixer.php
```

### 自动修复

```bash
php-cs-fixer fix /path/to/project
```

## 集成到项目

### 1. 安装 PHP CS Fixer

```bash
composer require --dev friendsofphp/php-cs-fixer
```

### 2. 创建配置文件 `.php-cs-fixer.php`

参考 `assets/.php-cs-fixer.php` 模板

### 3. 添加到 Composer

```json
{
    "scripts": {
        "cs:check": "php-cs-fixer fix --dry-run --diff",
        "cs:fix": "php-cs-fixer fix"
    }
}
```

## 参考资料

- [PSR-12 官方文档](https://www.php-fig.org/psr/psr-12/)
- [PHP 官方文档](https://www.php.net/manual/zh/)
- [PSR 标准系列](https://www.php-fig.org/psr/)

## 常见问题

**Q: 如何处理旧项目？**  
A: 逐步迁移，先修复新代码，再逐步处理旧代码。

**Q: 团队成员有不同偏好怎么办？**  
A: 使用统一的配置文件，所有人在同一套规范下工作。

**Q: 可以自定义规则吗？**  
A: 可以，在 `.php-cs-fixer.php` 中添加自定义规则。

## 工具推荐

- **PHP CS Fixer** - 代码格式化
- **PHPStan** - 静态分析
- **PHPMD** - 代码复杂度分析
- **PHPCS** - 代码风格检查
- **Rector** - 代码重构
