---
type: Reference
status: Organized
Belongs to: "[[MySQL]]"
Related to:
  - "[[mysql-视图存储过程与索引]]"
Has:
  - "[[慢查询日志]]"
  - "[[InnoDB]]"
  - "[[主从复制]]"
source: day23.md
created: 2026-06-14
updated: 2026-06-14
---

# MySQL 慢查询与主从复制

## 慢查询日志

```ini
[mysqld]
slow_query_log=ON
long_query_time=2
slow_query_log_file=/data/mysql-slow/slow.log
log_queries_not_using_indexes=ON
log_throttle_queries_not_using_indexes=10
```

| 参数 | 作用 |
| --- | --- |
| `long_query_time` | 记录执行时间大于该值的 SQL |
| `log_queries_not_using_indexes` | 记录未使用索引的 SQL |
| `log_throttle_queries_not_using_indexes` | 限制每分钟记录数量 |

日志切割：

```bash
mv /data/mysql-slow/slow.log /data/mysql-slow/slow.log.$(date +%F_%H%M%S)
mysqladmin -uroot -p flush-logs slow
```

分析工具：

```bash
mysqldumpslow -s t -t 10 /data/mysql-slow/slow.log
pt-query-digest /data/mysql-slow/slow.log > slow_report.txt
```

## 存储引擎

| 引擎 | 特点 | 场景 |
| --- | --- | --- |
| InnoDB | 默认，支持事务、行锁、崩溃恢复 | 绝大多数业务表 |
| MyISAM | 表锁，不支持事务 | 老系统兼容 |
| Memory | 数据存内存，重启丢失 | 临时高速表 |

## InnoDB 关键参数

| 参数 | 建议 |
| --- | --- |
| `innodb_buffer_pool_size` | 物理内存 50%-80% |
| `innodb_flush_log_at_trx_commit` | `1` 最安全，`2` 性能折中 |
| `innodb_log_file_size` | 较大提升写性能 |
| `innodb_file_per_table` | 建议开启 |

### 常用配置基线

```ini
[mysqld]
max_connections=600
max_allowed_packet=32M
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci

server-id=1
log-bin=/data/mysql-bin/binlog
binlog_format=ROW
binlog_expire_logs_seconds=604800

innodb_buffer_pool_size=4G
innodb_flush_log_at_trx_commit=2
innodb_log_file_size=128M
innodb_file_per_table=1
```

## binlog

```ini
[mysqld]
server-id=1
log-bin=/data/mysql-bin/binlog
binlog_format=ROW
```

```sql
SHOW MASTER STATUS;
```

### mysqlbinlog

| 参数 | 作用 |
| --- | --- |
| `--start-position` | 指定起始 position |
| `--stop-position` | 指定结束 position |
| `-d dbname` | 只解析指定库 |
| `--skip-gtids` | 跳过 GTID 信息 |

```bash
mysqlbinlog --no-defaults -d swk --skip-gtids binlog.000002 -r swk.sql
```

## 主从复制

主库记录 binlog → 从库 I/O 线程拉取并写入 relay log → 从库 SQL 线程回放。

### 主库配置

```ini
[mysqld]
server-id=1
log-bin=/data/mysql-bin/binlog
binlog_format=ROW
```

```sql
CREATE USER 'rep'@'%' IDENTIFIED BY '复杂密码';
GRANT REPLICATION SLAVE ON *.* TO 'rep'@'%';
```

### 从库配置

```ini
[mysqld]
server-id=2
```

```sql
CHANGE MASTER TO
  MASTER_HOST='192.168.31.10',
  MASTER_PORT=3306,
  MASTER_USER='rep',
  MASTER_PASSWORD='复杂密码',
  MASTER_LOG_FILE='binlog.000001',
  MASTER_LOG_POS=154;

START SLAVE;
SHOW SLAVE STATUS\G
```

### 状态检查

| 字段 | 正常值 | 含义 |
| --- | --- | --- |
| `Slave_IO_Running` | Yes | 是否能拉日志 |
| `Slave_SQL_Running` | Yes | 是否能回放 |
| `Seconds_Behind_Master` | 接近 0 | 主从延迟 |

### 延迟常见原因

从库太多、从库硬件弱、主库写入压力大、大事务、慢 SQL、网络抖动。

## 易错点

| 问题 | 处理 |
| --- | --- |
| 慢查询日志无限增长 | 定期切割 + flush-logs |
| `max_connections` 盲目调大 | 同时评估内存和线程 |
| 从库 `server-id` 重复 | 每台 MySQL 唯一 |
| 主从状态只看一个 Yes | I/O 和 SQL 两个线程都要正常 |
