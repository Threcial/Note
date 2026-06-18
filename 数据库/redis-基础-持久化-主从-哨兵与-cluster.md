---
type: Reference
status: Organized
source: day24.md
created: 2026-06-14
updated: 2026-06-14
Belongs to: "[[Redis]]"
Related to:
  - "[[高可用]]"
Has:
  - "[[Redis String]]"
  - "[[RDB]]"
  - "[[AOF]]"
  - "[[Redis Sentinel]]"
  - "[[Redis Cluster]]"
---
# Redis 基础、持久化、主从、哨兵与 Cluster

Redis 是基于内存的 Key-Value 存储系统，常用于缓存、计数器、排行榜、分布式锁、限流、发布订阅和会话状态存储。Redis 速度快的核心原因是主要操作在内存中完成，但也因此必须认真处理持久化和高可用。

这部分保留 Redis 安装、配置、基础数据类型、RDB/AOF、主从复制、哨兵、Cluster 的完整链路，并补充生产环境中更常见的安全、systemd、`SCAN`、`INFO` 和集群端口注意点。

## 关系整理

| 层次   | 组件                        | 作用                 |
| ---- | ------------------------- | ------------------ |
| 单机   | redis-server / redis-cli  | 服务端和客户端            |
| 数据模型 | string/list/set/zset/hash | Redis 最常用五种基础类型    |
| 持久化  | RDB/AOF/混合持久化             | 防止内存数据因进程退出或断电全部丢失 |
| 复制   | replicaof/masterauth      | 数据备份和读写分离          |
| 高可用  | Sentinel                  | 自动故障转移             |
| 分片   | Cluster                   | 多 master 分摊数据和流量   |
| 服务管理 | systemd                   | 开机自启、统一启停和日志排查     |

## 源码安装

安装依赖：

```bash
yum install epel-release -y
yum clean all
yum makecache
yum install jemalloc-devel gcc-c++ gcc tcl-devel -y
```

下载、编译、安装：

```bash
mkdir -p /app
cd /app
wget https://mirrors.huaweicloud.com/redis/redis-5.0.7.tar.gz
tar zxvf redis-5.0.7.tar.gz
cd redis-5.0.7
make
make PREFIX=/app/redis install
```

如果编译失败，先清理旧编译结果：

```bash
make distclean
make
make PREFIX=/app/redis install
```

## yum 安装补充

CentOS 7 默认仓库 Redis 版本偏旧，可以使用 Remi 源安装更高版本：

```bash
yum install -y http://rpms.famillecollet.com/enterprise/remi-release-7.rpm
yum --enablerepo=remi list redis
yum --enablerepo=remi install redis -y
```

yum 安装后的配置文件通常是：

```bash
/etc/redis.conf
```

启动方式：

```bash
systemctl start redis
systemctl enable redis
systemctl status redis
```

## redis.conf 常用配置

```ini
bind 0.0.0.0
protected-mode no
port 6379
daemonize yes
pidfile /var/run/redis_6379.pid
loglevel notice
logfile /var/log/redis/redis.log
databases 16
requirepass 123456
maxmemory 6gb
maxmemory-policy allkeys-lru
```

| 参数                 | 说明                                          |
| ------------------ | ------------------------------------------- |
| `bind`             | 监听地址，远程连接时不能只绑定 127.0.0.1                   |
| `protected-mode`   | 保护模式，生产环境不建议简单关闭，应配合密码、防火墙和内网限制             |
| `daemonize`        | 是否后台运行，systemd 管理时要和 service 的 `Type` 对应    |
| `requirepass`      | 设置认证密码                                      |
| `maxmemory`        | Redis 可用最大内存                                |
| `maxmemory-policy` | 内存淘汰策略，缓存场景常用 `allkeys-lru` 或 `allkeys-lfu` |

Redis 6 以后支持 ACL，生产环境比单一 `requirepass` 更细，可以给不同应用分配不同用户和权限。

## 启动和连接

```bash
cd /app/redis/bin
./redis-server ../redis.conf
./redis-cli -h 192.168.31.60 -p 6379 -a 123456
```

更安全的交互方式是连接后再认证，避免密码出现在 shell history 中：

```bash
redis-cli -h 192.168.31.60 -p 6379
AUTH 123456
INFO
```

关闭 Redis：

```bash
./redis-cli -a 123456 shutdown
```

## 基础数据类型

### String

String 是最基础类型，一个 key 对应一个 value。

```redis
SET name xwx
GET name
MSET age 20 sex man
MGET age sex
EXPIRE name 30
TTL name
PERSIST name
TYPE name
DEL name
```

`KEYS *` 会扫描全部 key，生产环境容易阻塞 Redis，不建议在大实例上使用。排查时优先使用：

```redis
SCAN 0 MATCH user:* COUNT 100
```

### List

List 是有序字符串列表，适合队列、栈、简单消息缓冲。

```redis
LPUSH queue a
RPUSH queue b
LRANGE queue 0 -1
LPOP queue
RPOP queue
LLEN queue
```

### Set

Set 是无序不重复集合，适合去重、标签、交集计算。

```redis
SADD user:1:tags linux redis mysql
SMEMBERS user:1:tags
SISMEMBER user:1:tags redis
SCARD user:1:tags
SINTER set1 set2
SREM user:1:tags mysql
```

### Sorted Set

Zset 是带 score 的有序集合，典型场景是排行榜。

```redis
ZADD rank 100 user1 90 user2
ZRANGE rank 0 -1 WITHSCORES
ZREVRANGE rank 0 9 WITHSCORES
ZSCORE rank user1
ZINCRBY rank 10 user2
ZREM rank user1
```

### Hash

Hash 适合保存对象字段。

```redis
HSET user:1 name xwx age 20
HGET user:1 name
HMGET user:1 name age
HGETALL user:1
HKEYS user:1
HLEN user:1
HINCRBY user:1 age 1
```

## 持久化

Redis 数据主要在内存里，没有持久化时进程退出或宕机会丢数据。Redis 常见持久化方式是 RDB、AOF 和混合持久化。

### RDB

RDB 是快照，把某一时刻的数据集写入磁盘。

```ini
save 900 1
save 300 10
save 60 10000
dbfilename dump.rdb
dir /data/redis
```

手动触发：

```redis
BGSAVE
LASTSAVE
```

`SAVE` 会阻塞 Redis 主进程，线上尽量不用；`BGSAVE` 会 fork 子进程在后台生成快照。

RDB 优点是恢复快、文件紧凑、适合全量备份；缺点是可能丢失最近一次快照之后的数据，并且 fork 时对内存和延迟有影响。

### AOF

AOF 以追加日志方式记录写命令，Redis 重启时按日志重放恢复数据。

```ini
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
```

| 策略         | 说明                  |
| ---------- | ------------------- |
| `always`   | 每条写命令都刷盘，最安全，最慢     |
| `everysec` | 每秒刷盘一次，常用折中方案       |
| `no`       | 由操作系统决定刷盘，性能最好，风险最大 |

AOF 文件通常比 RDB 大，恢复速度也可能慢于 RDB，但数据丢失窗口更小。

### 混合持久化

Redis 4.0 以后支持 AOF 混合持久化：前半段用 RDB 快照，后半段用增量 AOF 日志。

```ini
aof-use-rdb-preamble yes
```

混合持久化兼顾 RDB 恢复速度和 AOF 数据完整性。开启时要同时理解当前实例是否已经启用 AOF。

## systemd 服务化

源码安装后可以手动写 service 文件：

```ini
[Unit]
Description=Redis
After=network.target

[Service]
Type=forking
ExecStart=/app/redis/bin/redis-server /app/redis/redis.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/app/redis/bin/redis-cli -a 123456 shutdown
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

保存到：

```bash
/usr/lib/systemd/system/redis.service
```

启用：

```bash
systemctl daemon-reload
systemctl enable redis.service
systemctl start redis.service
systemctl status redis.service
```

注意 `redis.conf` 中 `dir ./` 的相对路径。用 systemd 启动时工作目录可能不是 Redis 的 bin 目录，RDB/AOF 可能写到意外位置。建议改成绝对路径：

```ini
dir /app/redis/data
```

## 主从复制

从节点配置：

```ini
replicaof 192.168.31.60 6379
masterauth 123456
```

旧版本写法是 `slaveof`，Redis 5 以后推荐 `replicaof`。

验证：

```redis
INFO Replication
```

主从复制优点是配置简单、读写分离、数据有备份；缺点是主节点故障不能自动切换，需要 Sentinel 或 Cluster 解决。

## Sentinel 哨兵

Sentinel 在主从复制基础上实现自动故障转移。建议至少 3 个哨兵节点，数量保持奇数。

核心概念：

| 概念     | 含义                |
| ------ | ----------------- |
| SDOWN  | 单个哨兵主观认为节点宕机      |
| ODOWN  | 多个哨兵达成共识，客观判定节点宕机 |
| quorum | 需要多少个哨兵同意故障判断     |

`sentinel.conf` 示例：

```ini
port 26379
daemonize yes
sentinel monitor mymaster 192.168.31.60 6379 2
sentinel auth-pass mymaster 123456
sentinel down-after-milliseconds mymaster 3000
sentinel failover-timeout mymaster 18000
```

启动：

```bash
./redis-sentinel ../sentinel.conf
redis-cli -p 26379 INFO Sentinel
```

多网卡、NAT 或虚拟机网络环境里，如果哨兵之间识别到错误地址，需要配置 announce：

```ini
sentinel announce-ip 192.168.31.60
sentinel announce-port 26379
```

如果 Redis 主库也需要被从库正确发现，可配：

```ini
replica-announce-ip 192.168.31.60
replica-announce-port 6379
```

## Cluster 模式

Redis Cluster 提供分片和高可用。整个集群有 16384 个 slot，key 会通过哈希映射到 slot，再落到对应 master 节点。

基础配置：

```ini
cluster-enabled yes
cluster-node-timeout 15000
cluster-config-file nodes_6379.conf
```

每个 Redis Cluster 节点除了业务端口，还会开放集群总线端口，通常是业务端口 + 10000。例如 6379 对应 16379。防火墙必须同时放通业务端口和集群总线端口。

手工节点发现：

```redis
CLUSTER MEET 192.168.111.234 6379
CLUSTER NODES
```

配置从节点：

```redis
CLUSTER REPLICATE <master-node-id>
```

现代命令更推荐直接用：

```bash
redis-cli --cluster create   192.168.111.234:6379 192.168.111.234:6380 192.168.111.234:6381   192.168.111.234:6382 192.168.111.234:6383 192.168.111.234:6384   --cluster-replicas 1
```

客户端访问 Cluster 时需要支持重定向。命令行加 `-c`：

```bash
redis-cli -c -h 192.168.111.234 -p 6379
```

否则遇到不属于当前节点的 slot，会看到 `MOVED` 提示。

## 常用排查命令

```redis
INFO
INFO Replication
INFO Persistence
INFO Memory
DBSIZE
CLIENT LIST
CONFIG GET dir
CONFIG GET appendonly
```

```bash
ss -lntp | grep redis
ps -ef | grep redis
journalctl -u redis -xe
redis-cli --bigkeys
redis-cli --latency
```

## 易错点

| 问题               | 处理                                |
| ---------------- | --------------------------------- |
| 远程无法连接           | 检查 `bind`、`protected-mode`、密码、防火墙 |
| `KEYS *` 卡住实例    | 大实例改用 `SCAN`                      |
| systemd 启动后数据不一致 | 检查 `dir` 是否是绝对路径                  |
| AOF/RDB 文件找不到    | 检查 `dir` 和进程启动方式                  |
| 哨兵互相发现不到         | 检查防火墙、端口、`announce-ip`            |
| Cluster 只放通 6379 | 还要放通 16379 等集群总线端口                |
| 从库连不上主库          | 检查 `masterauth`、主库密码和网络           |
