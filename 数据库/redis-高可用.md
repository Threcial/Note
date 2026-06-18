---
type: Concept
status: Organized
Belongs to: "[[Redis]]"
Related to:
  - "[[redis-基础与数据类型]]"
  - "[[高可用]]"
Has:
  - "[[Redis Sentinel]]"
  - "[[Redis Cluster]]"
source: day24.md
created: 2026-06-14
updated: 2026-06-14
---

# Redis 高可用

## 主从复制

从节点配置：

```ini
replicaof 192.168.31.60 6379
masterauth 123456
```

验证：

```redis
INFO Replication
```

主从复制配置简单，但主节点故障不能自动切换。

## Sentinel 哨兵

在主从基础上实现自动故障转移。建议至少 3 个哨兵节点，数量保持奇数。

| 概念 | 含义 |
| --- | --- |
| SDOWN | 单个哨兵主观认为宕机 |
| ODOWN | 多个哨兵共识，客观判定宕机 |
| quorum | 需要多少个哨兵同意 |

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

多网卡 NAT 环境配置 announce：

```ini
sentinel announce-ip 192.168.31.60
sentinel announce-port 26379
```

## Cluster 模式

提供分片和高可用。16384 个 slot，key 通过哈希映射到 slot。

```ini
cluster-enabled yes
cluster-node-timeout 15000
cluster-config-file nodes_6379.conf
```

每个节点开放业务端口和集群总线端口（业务端口 + 10000）。防火墙必须同时放通。

```bash
redis-cli --cluster create \
  192.168.111.234:6379 192.168.111.234:6380 192.168.111.234:6381 \
  192.168.111.234:6382 192.168.111.234:6383 192.168.111.234:6384 \
  --cluster-replicas 1
```

客户端加 `-c` 支持重定向：

```bash
redis-cli -c -h 192.168.111.234 -p 6379
```

## 常用排查命令

```redis
INFO
INFO Replication
INFO Persistence
INFO Memory
DBSIZE
CLIENT LIST
```

```bash
ss -lntp | grep redis
journalctl -u redis -xe
redis-cli --bigkeys
redis-cli --latency
```

## 易错点

| 问题 | 处理 |
| --- | --- |
| 哨兵互相发现不到 | 检查防火墙、端口、`announce-ip` |
| Cluster 只放通 6379 | 还要放通集群总线端口 |
| 从库连不上主库 | 检查 `masterauth`、主库密码和网络 |
