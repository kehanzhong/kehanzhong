# Git 提交规范

> 基于 Conventional Commits + Angular 规范，适配个人/团队使用

---

## 1. 提交信息格式

```
<type>(<scope>): <subject>

<body>

<footer>
```

### 最小格式（日常开发）

```
<type>: <简短描述>
```

### 示例

```
feat: 添加用户登录功能
fix: 修复订单金额计算精度丢失
refactor: 重构支付回调处理逻辑
docs: 更新API接口文档
perf: 优化首页查询改为缓存读取
```

---

## 2. Type（类型）

| Type | 说明 | 使用场景 |
|------|------|---------|
| `feat` | 新功能 | 新增接口、页面、模块 |
| `fix` | 修复 Bug | 任何 bug 修复 |
| `docs` | 文档变更 | README、API 文档、注释 |
| `style` | 代码格式 | 空格、缩进、分号等（不影响逻辑） |
| `refactor` | 重构 | 不改变功能，改进代码结构 |
| `perf` | 性能优化 | 缓存、索引、SQL 优化等 |
| `test` | 测试 | 添加/修改测试用例 |
| `chore` | 杂项 | 构建工具、依赖更新、配置文件 |
| `ci` | CI/CD | 持续集成/部署相关 |
| `build` | 构建 | 编译、打包等构建系统 |
| `revert` | 回滚 | 回滚之前的提交 |

---

## 3. Scope（范围，可选）

表示影响的范围，一般用模块名或文件名：

```
feat(auth): 增加JWT刷新机制
fix(order): 修复库存扣减并发问题
refactor(payment): 统一支付回调处理
perf(home): 首页接口合并减少请求数
```

### 常用 Scope 示例

| Scope | 含义 |
|-------|------|
| `auth` | 认证授权 |
| `user` | 用户模块 |
| `order` | 订单模块 |
| `payment` | 支付模块 |
| `api` | 接口层 |
| `admin` | 管理后台 |
| `mobile` | 移动端 |
| `config` | 配置 |
| `db` | 数据库 |

---

## 4. Subject（主题）

**规则：**

- 使用中文或英文，保持一致
- 长度不超过 72 字符
- 使用祈使语气（动词开头）
- 结尾不加句号
- 描述准确，见名知义

```
✅ 好的
feat: 增加用户手机号登录
fix: 修复导出Excel内存溢出
refactor: 简化订单创建流程

❌ 不好的
feat: 更新                       // 太模糊，不知道更新了什么
fix: 修了一个bug                  // 什么bug？
feat: 用户登录功能已完成。        // 不要句号，不要写"已完成"
```

---

## 5. Body（正文，可选）

用于详细说明 **为什么** 和 **做了什么**：

```
fix(order): 修复并发下单库存超卖

通过悲观锁（SELECT ... FOR UPDATE）替代乐观锁，
解决高并发场景下库存扣减不一致的问题。

调整内容：
- 下单前对表加写锁
- 减少事务锁持有时间
- 增加库存不足快速返回
```

---

## 6. Footer（页脚，可选）

### 关闭 Issue

```
fix: 修复分页异常

Closes #128
Refs #64
```

### Breaking Change（不兼容变更）

```
feat(api): 用户接口v2升级

BREAKING CHANGE: /api/v1/users 已废弃，
请迁移到 /api/v2/users，新接口返回结构有变化。
```

或者在 subject 后加 `!`：

```
feat(api)!: 用户接口v2升级
```

---

## 7. 提交粒度

### 一次提交 = 一件事

```
✅ 好的提交
feat: 增加订单导出功能
fix: 修复导出Excel内存溢出
test: 增加订单导出单元测试

❌ 大杂烩提交
feat: 增加订单导出、修复内存溢出、更新文档、重构导出服务、
      增加测试、更新依赖...
```

### 频繁提交优于大提交

- 每个逻辑变更独立提交
- 原子性：一个提交只做一件事
- 可回滚：出问题可以精确回滚，不影响其他改动

---

## 8. 分支命名

```
<type>/<描述>

feature/user-login          # 新功能
bugfix/order-amount-fix     # Bug修复
hotfix/payment-security     # 紧急修复
refactor/cleanup-service    # 重构
release/v2.1.0              # 发布
```

---

## 9. 工作流示例

```bash
# 1. 日常开发
git add app/Http/Controllers/UserController.php
git commit -m "feat(user): 增加用户头像上传"

# 2. 修复 Bug
git add app/Services/OrderService.php
git commit -m "fix(order): 修复并发下单超卖问题

通过悲观锁替代乐观锁，解决高并发下库存扣减不一致" -m "Closes #245"

# 3. 批量相关变更
git add database/migrations/ config/cache.php
git commit -m "perf: 首页查询替换为缓存

- 数据库查询改为 Redis 缓存
- 缓存过期时间设为 5 分钟
- 新增 cache:home:flush 命令"
```

---

## 10. 辅助工具

### Commitizen（交互式提交）

```bash
npm install -g commitizen cz-conventional-changelog
# 使用 git cz 代替 git commit
```

### Commitlint（提交检查）

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor',
      'perf', 'test', 'chore', 'ci', 'build', 'revert'
    ]],
    'subject-max-length': [2, 'always', 72],
    'subject-case': [0],
  },
};
```

### 安装 Git Hook

```bash
npx husky add .husky/commit-msg 'npx --no-install commitlint --edit "$1"'
```

---

## 11. 快速参考

```
feat:       新功能
fix:        Bug修复
docs:       文档
refactor:   重构
perf:       性能
test:       测试
chore:      杂项
revert:     回滚

格式: <type>(<scope>): <描述>
示例: feat(auth): 增加微信登录
```

---

## 12. 反面教材 ❌

```
wip                           # 无意义的提交信息
fix bug                       # 什么bug？
update                        # 更新了什么？
123                           # ？？？
save                          # 保存了什么？
临时提交                       # 请用 git stash
修复了一些问题                  # 哪些问题？
修复                                   # 一个词不够
```

## 正面示例 ✅

```
feat: 增加短信验证码登录
fix: 修复首页Banner图加载失败
refactor: 拆分UserService为多个职责单一服务
perf: 商品列表查询增加联合索引
docs: 补充支付接口错误码说明
test: 增加订单退款场景测试
chore: 升级PHP依赖到最新兼容版本
```

---

> 📝 **一个好的提交信息 = 半年后你能一眼看懂当时改了什么、为什么改。**
