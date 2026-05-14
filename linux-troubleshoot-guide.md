# Linux 排障速查手册

> 从负载到日志，生产环境出问题时的第一反应

---

## 目录

1. [性能排查速查](#1-性能排查速查)
2. [CPU 排查](#2-cpu-排查)
3. [内存排查](#3-内存排查)
4. [磁盘排查](#4-磁盘排查)
5. [网络排查](#5-网络排查)
6. [进程排查](#6-进程排查)
7. [日志排查](#7-日志排查)
8. [PHP-FPM 专查](#8-php-fpm-专查)
9. [MySQL 专查](#9-mysql-专查)
10. [Nginx 专查](#10-nginx-专查)
11. [Redis 专查](#11-redis-专查)
12. [应急场景](#12-应急场景)

---

## 1. 性能排查速查

```bash
# ═══ 1 分钟快速定位 ═══

uptime                       # 负载概况
dmesg -T | tail -20          # 内核日志（OOM/硬件错误）
vmstat 1 5                   # 系统整体：r(运行) b(阻塞) si/so(换入换出)
mpstat -P ALL 1 3            # 每个 CPU 核心使用率
pidstat 1 5                  # 每个进程的 CPU/内存/IO
iostat -xz 1 3               # 磁盘 IO 利用率 %util 接近100% = 磁盘瓶颈
free -h                      # 内存概况
sar -n DEV 1 3               # 网络吞吐量
sar -n TCP,ETCP 1 3          # TCP 连接状态

# ═══ top 技巧 ═══
top -c -o %CPU               # 按 CPU 排序
# 进入 top 后：
# 1    → 查看每个 CPU 核心
# c    → 显示完整命令行
# M    → 按内存排序
# P    → 按 CPU 排序
# H    → 线程模式（找 Java/Swoole 线程）

# ═══ htop（更友好） ═══
htop -p $(pgrep -d, php-fpm)
```

---

## 2. CPU 排查

```bash
# ═══ 整体 ═══
top -bn1 | head -20          # 一次性输出
mpstat -P ALL 1               # 每核使用率

# ═══ 找 CPU 最高的进程 ═══
ps aux --sort=-%cpu | head -5

# ═══ 找 CPU 最高的线程 ═══
ps -Lp <PID> -o pid,tid,pcpu,comm --sort=-pcpu | head -10
# 或
top -H -p <PID>

# ═══ 线程栈追踪（适合卡死排查） ═══
# 安装 perf/bpftrace
perf top -p <PID>             # 实时看函数 CPU 占用
strace -p <PID> -c -f          # 统计系统调用耗时

# ═══ CPU 负载解读 ═══
# uptime 输出: 14:30:01 up 30 days, 3 users, load average: 5.2, 4.1, 3.8
# load average < CPU 核数 → 正常
# load average > CPU 核数 × 1.5 → 压力大
# load average > CPU 核数 × 3 → 严重过载

nproc  # 看核数
```

---

## 3. 内存排查

```bash
# ═══ 整体 ═══
free -h
#             total   used   free   shared   buff/cache   available
# Mem:        7.7G    3.2G   1.1G   500M     3.4G         4.0G
#
# available < 10% total → 内存不足

# ═══ 是什么用掉了内存 ═══
# 按 RSS（实际物理内存）排序
ps aux --sort=-rss | head -20

# 按进程统计
ps -eo pid,user,rss,comm --sort=-rss | head -20

# PHP-FPM 每个进程的内存
ps -C php-fpm -o pid,rss,args --no-headers | awk '{sum+=$2; count++} END {print "平均:" sum/count/1024 "MB 总数:" count}'

# ═══ OOM 排查 ═══
dmesg -T | grep -i "out of memory"
dmesg -T | grep -i "killed process"
# 看是什么进程被杀，当时用了多少内存

# ═══ 文件缓存（不是真正的"占用"） ═══
# Linux 会拿空闲内存当文件缓存（buffer/cache）
# 这是好事，不是问题！available 才是真实的可用内存
# 用 free -h 看 available 而非 free

# ═══ 手动释放缓存（紧急使用，有性能影响） ═══
sync && echo 3 > /proc/sys/vm/drop_caches
```

---

## 4. 磁盘排查

```bash
# ═══ 磁盘空间 ═══
df -h                        # 各分区容量
df -i                        # inode 使用率（inode用完=不能建新文件）
du -sh /* 2>/dev/null | sort -rh | head -10  # 根目录下谁最大

# ═══ 找大文件 ═══
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null
find /var/log -type f -name "*.log" -size +1G

# ═══ IO 性能 ═══
iostat -xz 1
# %util 接近 100% → 磁盘是瓶颈
# await >> svctm → IO 排队严重
# r/s + w/s = IOPS

# ═══ 是哪个进程在狂读写 ═══
iotop -o                     # 只看有 IO 的进程
# 或
pidstat -d 1

# ═══ 安全清理 ═══
journalctl --vacuum-size=500M      # 清理 systemd 日志
docker system prune -a -f          # 清理 Docker
apt-get clean                      # 清理 apt 缓存
# 清理旧内核
dpkg -l | grep linux-image
apt-get autoremove --purge
```

---

## 5. 网络排查

```bash
# ═══ 连通性 ═══
ping -c 4 google.com         # 通不通
traceroute google.com        # 经过的路由
mtr google.com               # 持续路由追踪（动态更新）

# ═══ DNS ═══
nslookup example.com         # 域名解析
dig example.com +short       # 看所有记录
cat /etc/resolv.conf          # DNS 配置

# ═══ 连接状态 ═══
ss -tunlp                    # 所有监听端口
ss -s                        # 连接统计
ss -antp state time-wait | wc -l   # TIME_WAIT 数量
ss -antp state established | wc -l # ESTABLISHED 数量

# ═══ 端口占用 ═══
lsof -i :80                  # 谁在用 80 端口
lsof -i :3306,6379           # 一起查多个

# ═══ 抓包 ═══
tcpdump -i eth0 port 80 -A -c 20    # 抓 HTTP 请求
tcpdump -i eth0 host 10.0.0.5 -w capture.pcap  # 保存到文件
# 高级：用 tshark 分析
tshark -r capture.pcap -Y "http.request" -T fields -e http.host -e http.request.uri

# ═══ TCP 连接排查 ═══
# SYN_SENT 多 → 连不上后端
ss -ant | awk '{print $1}' | sort | uniq -c | sort -rn

# 内核调优
sysctl net.ipv4.tcp_fin_timeout         # TIME_WAIT 时间
sysctl net.ipv4.tcp_tw_reuse            # 复用 TIME_WAIT
sysctl net.core.somaxconn               # 最大连接队列

# 临时调
sysctl -w net.ipv4.tcp_tw_reuse=1
```

---

## 6. 进程排查

```bash
# ═══ 进程树 ═══
pstree -p                    # 看父子关系
pstree -p $(pgrep nginx)     # Nginx 的 master+worker

# ═══ 进程环境变量 ═══
cat /proc/<PID>/environ | tr '\0' '\n'  # 看进程启动时环境变量

# ═══ 进程打开的文件 ═══
lsof -p <PID>                # 所有打开的文件/连接/socket
lsof -p <PID> | grep php     # 只看 PHP 相关的

# ═══ 进程限制 ═══
cat /proc/<PID>/limits       # 文件描述符/内存等限制
ulimit -n                    # 当前 fd 上限

# ═══ 杀死进程 ═══
kill -TERM <PID>             # 优雅终止（推荐）
kill -9 <PID>                # 强制杀（进程状态变僵尸时用）
killall -TERM php-fpm        # 按名称杀
pkill -f "php artisan queue:work"  # 按命令行匹配

# ═══ 线程查看 ═══
ps -eLf | grep <PID> | wc -l  # 线程数
```

---

## 7. 日志排查

```bash
# ═══ tail 技巧 ═══
tail -f /var/log/nginx/error.log           # 实时跟踪
tail -n 100 /var/log/nginx/access.log     # 最后 100 行
tail -f /var/log/nginx/access.log | grep " 500 "  # 只看 500

# ═══ 搜索 ═══
grep "2026-05-14 09:" /var/log/nginx/access.log | wc -l  # 今天9点的请求数
grep " 502 " /var/log/nginx/error.log | tail -20
grep -B5 -A5 "Fatal error" /var/log/app.log    # 错误前后5行

# ═══ 统计 ═══
# 访问最多的 URL TOP 10
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# 状态码分布
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# 响应时间超过 1 秒的请求
awk '$NF > 1' /var/log/nginx/access.log

# ═══ 日志切割 ═══
logrotate -f /etc/logrotate.d/nginx   # 强制切割
# 或手动
mv access.log access.log.$(date +%Y%m%d)
kill -USR1 $(cat /var/run/nginx.pid)  # Nginx 重开日志文件

# ═══ journalctl（systemd 服务） ═══
journalctl -u nginx -f           # 跟踪
journalctl -u php8.2-fpm --since "10 minutes ago"
journalctl -u php8.2-fpm --since today | grep -i error
```

---

## 8. PHP-FPM 专查

```bash
# ═══ 状态 ═══
systemctl status php8.2-fpm
ps aux | grep php-fpm | wc -l        # 进程数

# ═══ 慢日志 ═══
tail -100 /var/log/php-fpm-slow.log
# 看哪些脚本慢，为什么慢（堆栈）

# ═══ 错误日志 ═══
tail -100 /var/log/php8.2-fpm.log | grep -E "ERROR|Fatal|Exception"
# WARNING: pool www server reached pm.max_children → 进程不够

# ═══ 连不上 FPM ═══
# 1. 检查监听
ss -tlnp | grep 9000
ls -la /run/php/php8.2-fpm.sock      # socket 方式

# 2. 是否进程满了
# 看 listen queue（在 /status 页面）
curl http://localhost/status?full

# 3. 手动测试连接
echo SCRIPT_NAME=/ping SCRIPT_FILENAME=/ping QUERY_STRING= REQUEST_METHOD=GET | cgi-fcgi -bind -connect 127.0.0.1:9000

# ═══ 进程数不够信号 ═══
# pool www seems busy / max children reached
# → 调大 pm.max_children
# 或排查是否有慢脚本堵住进程
```

---

## 9. MySQL 专查

```bash
# ═══ 连接 ═══
mysql -u root -p -e "SHOW PROCESSLIST"
mysql -u root -p -e "SHOW FULL PROCESSLIST" | grep -v Sleep | wc -l

# ═══ 锁 ═══
mysql -u root -p -e "SELECT * FROM information_schema.innodb_locks"
mysql -u root -p -e "SELECT * FROM information_schema.innodb_lock_waits"
# 找到等待的事务，杀掉
mysql -u root -p -e "KILL <connection_id>"

# ═══ 慢查询 ═══
# 临时开启
mysql -u root -p -e "SET GLOBAL slow_query_log = 1"
mysql -u root -p -e "SET GLOBAL long_query_time = 2"
# 查看
tail -100 /var/log/mysql/slow-query.log
pt-query-digest /var/log/mysql/slow-query.log  # 分析

# ═══ 连接数 ═══
mysql -u root -p -e "SHOW VARIABLES LIKE 'max_connections'"
mysql -u root -p -e "SHOW STATUS LIKE 'Threads_connected'"
# Threads_connected / max_connections > 80% → 危险

# ═══ 表状态 ═══
mysqlcheck -u root -p --all-databases      # 检查所有表
mysql -u root -p -e "SHOW TABLE STATUS LIKE 'orders'"
```

---

## 10. Nginx 专查

```bash
# ═══ 配置验证 ═══
nginx -t
nginx -T | grep -A10 "location /api"

# ═══ 502/504 排查 ═══
# 502: Nginx 连不上后端（PHP-FPM 挂了/进程满了）
# 检查：systemctl status php8.2-fpm
# 检查：ps aux | grep php-fpm | wc -l
# 检查：/var/log/php8.2-fpm.log

# 504: 后端响应超时
# 检查：fastcgi_read_timeout 是否够
# 检查：PHP 慢日志，是否脚本执行超过超时时间

# ═══ 请求量 ═══
# 实时 QPS
tail -f /var/log/nginx/access.log | pv -l -i1 > /dev/null
# 或
while true; do echo "$(date): $(tail -10000 /var/log/nginx/access.log | wc -l)/10s"; sleep 10; done

# ═══ 错误分布 ═══
awk '($9>=500){print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# ═══ 连接队列溢出 ═══
# listen queue overflow
# → 增大 net.core.somaxconn + nginx backlog
sysctl -w net.core.somaxconn=65535
```

---

## 11. Redis 专查

```bash
# ═══ 连接 ═══
redis-cli ping
redis-cli info clients         # 连接客户端数
redis-cli client list          # 连接详情

# ═══ 内存 ═══
redis-cli info memory | grep used_memory_human
redis-cli info memory | grep maxmemory_human
redis-cli MEMORY DOCTOR        # 内存分析建议

# ═══ 慢查询 ═══
redis-cli slowlog get 10       # 最近 10 条慢查询
redis-cli slowlog reset        # 清空

# ═══ 键分析 ═══
redis-cli --bigkeys            # 找大 key（小心阻塞！用 --hotkeys 代替）
redis-cli DBSIZE               # 总键数
redis-cli info keyspace        # 各 DB 键数 + 对应过期数

# ═══ 性能 ═══
redis-cli --latency            # 延迟监控
redis-cli --latency-history    # 历史延迟
redis-cli --intrinsic-latency 100  # 本机基准延迟

# ═══ 复制 ═══
redis-cli info replication     # 主从状态
# master_link_status:up → 正常
# master_link_status:down → 同步断了
```

---

## 12. 应急场景

### 12.1 CPU 100%

```bash
# 1. 找进程
top -o %CPU
# 2. 看是什么操作
strace -p <PID> -f -e trace=all 2>&1 | head -50
# 3. 如果是 PHP，看慢日志
tail -50 /var/log/php-fpm-slow.log
# 4. 如果是 MySQL，看 ProcessList
mysql -u root -p -e "SHOW FULL PROCESSLIST"
# 5. 临时止损
kill -STOP <PID>   # 暂停进程（留活口分析）
# 分析完再 kill -CONT <PID> 或 kill -TERM <PID>
```

### 12.2 OOM（内存耗尽）

```bash
dmesg -T | tail -30 | grep -i oom
# Out of memory: Kill process XXXX (php-fpm) score 850
# → PHP-FPM 吃太多内存

# 止损
systemctl restart php8.2-fpm

# 排查
# 1. 调小 pm.max_children
# 2. 检查是否有内存泄漏（pm.max_requests设小）
# 3. 检查是否有超大上传/处理
```

### 12.3 磁盘满

```bash
df -h /           # 看哪个分区满了
du -sh /* 2>/dev/null | sort -rh | head -5  # 找出大户

# 常见原因
# /var/log 日志 → logrotate -f
# /tmp session 文件 → find /tmp -mtime +7 -delete
# Docker overlay → docker system prune -a
# MySQL binlog → PURGE BINARY LOGS
# Core dump → rm /var/coredump/*
```

### 12.4 网站 502

```bash
# 1. PHP-FPM 活着吗
systemctl status php8.2-fpm
# 2. 进程数满没满
curl http://localhost/status?full | grep "max children reached"
# 3. 重启
systemctl restart php8.2-fpm
# 4. 长期方案
# 增大 pm.max_children / pm.max_requests
```

### 12.5 TIME_WAIT 爆满

```bash
ss -s  # 看连接统计
# 大量 TIME_WAIT 正常（短连接多），但几万+ 说明有问题

# 临时
sysctl -w net.ipv4.tcp_tw_reuse=1

# 永久方案：改为长连接
# Nginx: upstream { keepalive 32; }
# Nginx: proxy_set_header Connection "";
# PHP: 客户端复用连接
```

---

> 🔧 **排障不是凭感觉，是数据驱动的。先看指标，再定位根因，最后才是修复。**
