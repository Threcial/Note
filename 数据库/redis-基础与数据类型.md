---
type: Reference
status: Organized
Belongs to: "[[Redis]]"
Has:
  - "[[Redis String]]"
  - "[[Redis List]]"
  - "[[Redis Set]]"
  - "[[Redis Zset]]"
  - "[[Redis Hash]]"
  - "[[RDB]]"
  - "[[AOF]]"
source: day24.md
created: 2026-06-14
updated: 2026-06-14
---

# Redis 基础与数据类型

## 安装

```bash
yum install jemalloc-devel gcc-c++ gcc tcl-devel -y

wget https://mirrors.huaweicloud.com/redis/redis-5.0.7.tar.gz
tar zxvf redis-5.0.7.tar.gz
cd redis-5.0.7
make
make PREFIX=/app/redis install
```

## redis.conf 常用配置

```ini
bind 0.0.0.0
port 6379
daemonize yes
loglevel notice
logfile /var/log/redis/redis.log
requirepass 123456
maxmemory 6gb
maxmemory-policy allkeys-lru
```

| 参数 | 说明 |
| --- | --- |
| `requirepass` | 认证密码 |
| `maxmemory` | 可用最大内存 |
| `maxmemory-policy` | 淘汰策略，缓存场景常用 `allkeys-lru` |

## 启动与连接

```bash
./redis-server ../redis.conf
redis-cli -h 192.168.31.60 -p 6379 -a 123456
```

更安全：连接后再认证，避免密码出现在 history 中：

```bash
redis-cli -h 192.168.31.60 -p 6379
AUTH 123456
```

## 基础数据类型

### String

```redis
SET name xwx
GET name
MSET age 20 sex man
MGET age sex
EXPIRE name 30
TTL name
DEL name
```

生产环境避免使用 `KEYS *`，改用 `SCAN`：

```redis
SCAN 0 MATCH user:* COUNT 100
```

### List

```redis
LPUSH queue a
RPUSH queue b
LRANGE queue 0 -1
LPOP queue
RPOP queue
```

### Set

```redis
SADD user:1:tags linux redis mysql
SMEMBERS user:1:tags
SISMEMBER user:1:tags redis
SINTER set1 set2
```

### Sorted Set

```redis
ZADD rank 100 user1 90 user2
ZRANGE rank 0 -1 WITHSCORES
ZREVRANGE rank 0 9 WITHSCORES
ZINCRBY rank 10 user2
```

### Hash

```redis
HSET user:1 name xwx age 20
HGET user:1 name
HGETALL user:1
HINCRBY user:1 age 1
```

## 持久化

### RDB

```ini
save 900 1
save 300 10
save 60 10000
dbfilename dump.rdb
```

手动触发：`BGSAVE`（后台），`SAVE`（阻塞主进程）。

| 特点 | 说明 |
| --- | --- |
| 优点 | 恢复快、文件紧凑、适合全量备份 |
| 缺点 | 可能丢最近一次快照后数据 |

### AOF

```ini
appendonly yes
appendfsync everysec
```

| 策略 | 说明 |
| --- | --- |
| `always` | 每条写命令刷盘，最安全最慢 |
| `everysec` | 每秒刷盘，常用折中 |
| `no` | 操作系统决定刷盘，性能最好风险最大 |

### 混合持久化（Redis 4.0+）

```ini
aof-use-rdb-preamble yes
```

兼顾 RDB 恢复速度和 AOF 数据完整性。

## systemd 服务化

```ini
[Unit]
Description=Redis
After=network.target

[Service]
Type=forking
ExecStart=/app/redis/bin/redis-server /app/redis/redis.conf
ExecStop=/app/redis/bin/redis-cli -a 123456 shutdown

[Install]
WantedBy=multi-user.target
```

注意 `redis.conf` 中的 `dir` 应使用绝对路径。

## 易错点

| 问题 | 处理 |
| --- | --- |
| 远程无法连接 | 检查 `bind`、`protected-mode`、密码、防火墙 |
| `KEYS *` 卡住实例 | 大实例改用 `SCAN` |
| systemd 启动后数据不一致 | 检查 `dir` 是否是绝对路径 |
