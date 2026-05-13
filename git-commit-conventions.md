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

## 7. 基本操作速查

### 7.1 工作区流转

```
工作区 (Working Directory)
    │  git add
    ▼
暂存区 (Staging Area)
    │  git commit
    ▼
本地仓库 (Local Repo)
    │  git push
    ▼
远程仓库 (Remote Repo)
    │  git pull / git fetch
    ▼
工作区 ← 回到起点
```

```bash
# 查看状态
git status                    # 工作区 & 暂存区状态
git status -s                 # 精简输出
git diff                      # 工作区 vs 暂存区
git diff --staged             # 暂存区 vs HEAD
git diff HEAD                 # 工作区 vs HEAD
git diff <commitA> <commitB>  # 两个提交对比

# 查看日志
git log --oneline --graph --all  # 最常用的日志视图
git log -p <file>                 # 查看某文件的变更历史
git log --author="khz"           # 按作者筛选
git log --since="2026-05-01"     # 按时间筛选
git show <commit>                 # 查看某次提交详情
git blame <file>                  # 逐行追溯作者
```

### 7.2 暂存与恢复

```bash
# 暂存文件
git add <file>           # 添加指定文件
git add <file1> <file2>  # 添加多个
git add .                # 添加所有（当前目录及子目录）
git add -A               # 添加所有（包括删除）
git add -p               # 交互式选择片段

# 取消暂存
git restore --staged <file>   # 从暂存区移除（保留修改）
git reset HEAD <file>         # 同上（旧语法）

# 丢弃工作区修改
git restore <file>            # 丢弃单个文件修改
git restore .                 # 丢弃所有修改（危险⚠️）
git checkout -- <file>        # 同上（旧语法）
git clean -fd                 # 删除未跟踪的文件和目录
```

### 7.3 stash 暂存

```bash
git stash                    # 暂存当前改动（回到干净工作区）
git stash save "描述信息"    # 带描述的暂存
git stash list               # 查看暂存列表
git stash pop                # 弹出最近的暂存（恢复并删除记录）
git stash pop stash@{1}      # 弹出指定暂存
git stash apply              # 恢复但不删除记录
git stash drop stash@{0}     # 删除指定暂存
git stash clear              # 清空所有暂存

# 典型场景：开发到一半需要切分支
# ❌ 不会丢失进度
git stash
git switch feature-other
# ... 处理完事情 ...
git switch feature-original
git stash pop
```

---

## 8. 分支操作

### 8.1 分支查看

```bash
git branch              # 本地分支列表（* 标记当前）
git branch -r           # 远程分支列表
git branch -a           # 所有分支
git branch -v           # 分支最近提交
git branch --merged     # 已合并的分支
git branch --no-merged  # 未合并的分支
```

### 8.2 分支创建与切换

```bash
# 创建分支
git branch <branch>              # 基于当前分支创建
git branch <branch> <commit>     # 基于指定提交创建

# 切换分支（推荐 switch）
git switch <branch>              # 切换到已有分支
git switch -c <branch>           # 创建并切换（新语法）
git checkout -b <branch>         # 同上（旧语法）
git switch -                     # 切换回上一个分支

# 创建并关联远程分支
git switch -c feature/login
git push -u origin feature/login  # -u 建立跟踪关系

# 典型工作流
main ← feature/login ← feature/login-2fa
git switch main
git switch -c feature/login          # 从 main 创建
git switch -c feature/login-2fa       # 从 login 创建
```

### 8.3 分支删除

```bash
git branch -d <branch>        # 安全删除（已合并的才能删）
git branch -D <branch>        # 强制删除（不管合没合并）
git push origin -d <branch>   # 删除远程分支
```

### 8.4 分支重命名

```bash
git branch -m <old> <new>     # 重命名分支
git branch -m <new>           # 重命名当前分支
```

---

## 9. 合并与变基

### 9.1 merge（合并）

```bash
# 将 feature/login 合并到 main
git switch main
git merge feature/login

# 合并策略
# Fast-forward：main 无新提交，直接移动指针（线性历史）
#   A---B---C (main, feature)
#   结果：A---B---C (main, feature)
#
# 3-way merge：两个分支都有新提交，创建一个合并提交
#   A---B---D (main)
#        \
#         C---E (feature)
#   结果：A---B---D---M (main)   M = merge commit
#             \     /
#              C---E

git merge --no-ff feature/login    # 总是创建合并提交（推荐）
git merge --ff-only feature/login  # 仅允许快进合并
git merge --squash feature/login   # 压缩为一个提交（不保留分支历史）
git merge --abort                  # 放弃合并（冲突时）
```

### 9.2 rebase（变基）

```bash
# 将 feature 的改动「接到」main 最新提交之后
# 结果更线性，历史更干净
#
# 变基前:
#   A---B---D (main)
#        \
#         C---E (feature)
#
# git switch feature && git rebase main
# 变基后:
#   A---B---D (main)---C'---E' (feature)

git switch feature/login
git rebase main                 # 将当前分支变基到 main
git rebase --continue           # 解决冲突后继续
git rebase --skip               # 跳过当前提交
git rebase --abort              # 放弃变基，回到之前状态

# 交互式变基（整理提交历史）
git rebase -i HEAD~3            # 整理最近3个提交
# pick  → 保留该提交
# reword → 修改提交信息
# squash → 合并到上一个提交
# drop   → 删除该提交
# edit   → 暂停修改该提交
```

### 9.3 merge vs rebase

| 场景 | 推荐 | 原因 |
|------|------|------|
| 合并功能分支到 main | `merge --no-ff` | 保留完整分支历史 |
| 同步 main 最新代码到 feature | `rebase` | 保持线性历史 |
| 公开分支（别人也在用） | `merge` | rebase 会改写历史 |
| 整理本地未推送的提交 | `rebase -i` | 整理后一次推送 |

> ⚠️ **黄金规则：永远不要 rebase 已经推送到远程的分支！**

### 9.4 cherry-pick（摘樱桃）

```bash
# 把某个提交「复制」到当前分支
git cherry-pick <commit>
git cherry-pick <commitA>..<commitB>   # 复制一段提交（不含A）
git cherry-pick <commitA>^..<commitB>  # 复制一段提交（含A）
git cherry-pick --continue             # 解决冲突后继续
git cherry-pick --abort                # 放弃
```

---

## 10. 冲突解决

### 10.1 冲突标记

```
<<<<<<< HEAD
当前分支的代码
=======
要合并分支的代码
>>>>>>> feature/login
```

### 10.2 解决步骤

```bash
# 1. 冲突发生
git merge feature/login
# 输出：CONFLICT (content): Merge conflict in src/User.php

# 2. 查看冲突文件
git status                 # 查看哪些文件冲突
git diff                   # 查看冲突内容

# 3. 手动编辑冲突文件，移除标记，保留最终想要的代码
# <<<<<<< 和 ======= 和 >>>>>>> 这些行都要删掉

# 4. 标记为已解决
git add <file>             # 添加已解决的文件

# 5. 完成合并
git commit                 # 提交（不需要 -m，会自动生成 merge message）
# 或放弃
git merge --abort
```

### 10.3 实用技巧

```bash
# 以某一方为准
git checkout --ours <file>    # 保留当前分支的版本
git checkout --theirs <file>  # 保留合并分支的版本

# 查看冲突的三方对比
git diff :1:file :2:file :3:file
# :1 = 共同祖先
# :2 = 当前分支 (ours)
# :3 = 合并分支 (theirs)

# 用图形化工具解决冲突
git mergetool                   # 打开配置的合并工具

# 查看哪些提交在 A 但不在 B 中
git log --oneline main..feature    # feature 有但 main 没有的
git log --oneline feature..main    # main 有但 feature 没有的
```

### 10.4 冲突预防

```bash
# 合并前先拉取最新代码
git pull --rebase

# 频繁拉取，减少冲突概率
# 每天开始工作前：git pull
# 提交前：git pull --rebase

# 大改动前先同步
git fetch origin
git diff main origin/main    # 看看别人改了什么
```

---

## 11. 回滚操作

### 11.1 撤销提交

```bash
git revert <commit>           # 反向提交（安全，保留历史）
git revert HEAD               # 撤销最近一次提交
git revert --continue         # 解决冲突后继续

# reset（危险，改写历史）
git reset --soft HEAD~1       # 撤销提交，改动回暂存区
git reset --mixed HEAD~1      # 撤销提交+暂存，改动回工作区（默认）
git reset --hard HEAD~1       # 彻底丢弃改动 ⚠️ 不可恢复
```

### 11.2 场景对照

| 场景 | 命令 |
|------|------|
| 还没 add | `git restore <file>` |
| 已经 add 还没 commit | `git restore --staged <file>` |
| 已经 commit 还没 push | `git reset --soft HEAD~1` |
| 已经 push 到远程 | `git revert HEAD` + `git push` |
| 不小心 commit 到错误分支 | `git reset --soft HEAD~1` 然后 `git stash` |
| 找被删除的提交 | `git reflog` → `git cherry-pick` |

### 11.3 reflog（后悔药）

```bash
git reflog                     # 查看所有 HEAD 变更记录
git reflog show <branch>       # 查看某分支的历史
git checkout HEAD@{3}          # 跳转到 3 步前的状态
```

---

## 12. 远程协作

```bash
# 查看远程
git remote -v                         # 远程仓库列表
git remote add <name> <url>           # 添加远程
git remote remove <name>              # 删除远程

# 拉取
git fetch origin                      # 下载远程更新（不合并）
git pull                              # fetch + merge
git pull --rebase                     # fetch + rebase（推荐日常用）

# 推送
git push origin <branch>              # 推送到远程
git push -u origin <branch>           # 首次推送并建立跟踪
git push --force-with-lease           # 强制推送（比 --force 安全）
# ⚠️ 永远不要 git push --force 到共享分支！

# 标签
git tag                               # 列表
git tag v1.0.0                        # 轻量标签
git tag -a v1.0.0 -m "版本说明"       # 附注标签（含作者+时间）
git push origin v1.0.0                # 推送标签
git push origin --tags                # 推送所有标签
git tag -d v1.0.0                     # 删除本地标签
git push origin -d v1.0.0             # 删除远程标签
```

---

## 13. 提交粒度

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

## 14. 分支命名

```
<type>/<描述>

feature/user-login          # 新功能
bugfix/order-amount-fix     # Bug修复
hotfix/payment-security     # 紧急修复
refactor/cleanup-service    # 重构
release/v2.1.0              # 发布
```

---

## 15. 工作流示例

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

## 16. 辅助工具

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

## 17. 快速参考

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

## 18. 反面教材 ❌

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
