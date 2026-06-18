---
title: MySQL 慢查询、存储引擎、InnoDB 参数与主从复制
status: Organized
source: day23.md
created: 2026-06-14
updated: 2026-06-14
Belongs to: "[[MySQL]]"
Related to:
  - "[[MySQL 索引]]"
  - "[[MySQL 事务]]"
Has:
  - "[[慢查询日志]]"
  - "[[InnoDB]]"
  - "[[主从复制]]"
  - "[[relay log]]"
type: Reference
---
# MySQL 慢查询、存储引擎、InnoDB 参数与主从复制

这部分内容集中在 MySQL 运维侧：慢查询用于发现问题 SQL，InnoDB 参数决定大部分读写性能，主从复制依赖 binlog 和 relay log 完成数据同步

## 慢查询日志

慢查询日志记录执行时间超过阈值的 SQL。它不是性能优化本身，而是定位性能问题的入口。

```ini
[mysqld]
slow_query_log=ON
long_query_time=2
slow_query_log_file=/data/mysql-slow/slow.log
log_queries_not_using_indexes=ON
log_throttle_queries_not_using_indexes=10
```

| 参数                                       | 作用                   |
| ---------------------------------------- | -------------------- |
| `slow_query_log=ON`                      | 开启慢查询日志              |
| `long_query_time=2`                      | 记录执行时间大于 2 秒的 SQL    |
| `slow_query_log_file`                    | 指定慢查询日志路径            |
| `log_queries_not_using_indexes=ON`       | 记录没有使用索引的 SQL        |
| `log_throttle_queries_not_using_indexes` | 限制每分钟记录的未使用索引 SQL 数量 |

`log_queries_not_using_indexes` 不能无脑打开。业务高峰期如果大量 SQL 都没命中索引，慢查询日志会迅速膨胀，反过来制造磁盘 I/O 压力。配合 `log_throttle_queries_not_using_indexes` 更稳。

## 慢查询日志切割

MySQL 没有内置“按大小自动切割慢查询日志”的参数。常规做法是先移动旧日志，再通知 MySQL 重新打开日志文件。

```bash
mv /data/mysql-slow/slow.log /data/mysql-slow/slow.log.$(date +%F_%H%M%S)
mysqladmin -uroot -p flush-logs slow
```

如果直接用 Vim 打开慢日志并保存，可能会改变文件 inode 或文件状态，导致 MySQL 继续写旧文件句柄。规范动作是 `mv + flush slow logs`。

## 慢查询分析

系统自带工具是 `mysqldumpslow`。

```bash
mysqldumpslow -s t -t 10 /data/mysql-slow/slow.log
mysqldumpslow -s c -t 20 -g 'select' /data/mysql-slow/slow.log
```

| 参数         | 说明        |
| ---------- | --------- |
| `-s c`     | 按访问次数排序   |
| `-s l`     | 按锁等待时间排序  |
| `-s r`     | 按返回记录数排序  |
| `-s t`     | 按查询时间排序   |
| `-t N`     | 只显示前 N 条  |
| `-g regex` | 用正则过滤 SQL |

现代环境更常用 Percona Toolkit 的 `pt-query-digest`，它比 `mysqldumpslow` 的聚合维度更细，适合生产慢日志分析。

```bash
pt-query-digest /data/mysql-slow/slow.log > slow_report.txt
```

MySQL 8.0 也可以结合 `EXPLAIN ANALYZE` 检查实际执行计划：

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1001;
```

## 存储引擎

| 存储引擎   | 特点                      | 场景           |
| ------ | ----------------------- | ------------ |
| InnoDB | 默认引擎，支持事务、行锁、外键、崩溃恢复    | 绝大多数业务表      |
| MyISAM | 表锁，不支持事务，MySQL 5.5 之前常见 | 老系统兼容        |
| Memory | 数据存内存，重启丢失              | 临时高速表，实际使用较少 |

InnoDB 是当前主线。除非兼容历史系统，业务表默认按 InnoDB 设计。

## InnoDB 关键参数

```sql
SHOW VARIABLES LIKE 'innodb%';
```

| 参数                               | 常见设置思路                                |
| -------------------------------- | ------------------------------------- |
| `innodb_buffer_pool_size`        | 保存数据页和索引页，通常按物理内存 50%-80% 规划          |
| `innodb_flush_log_at_trx_commit` | `1` 最安全，`2` 常用于性能和可靠性折中，`0` 性能最好但风险更大 |
| `innodb_log_buffer_size`         | 默认 16M 一般够用                           |
| `innodb_log_file_size`           | 较大可提升写性能，但可能增加崩溃恢复时间                  |
| `innodb_log_files_in_group`      | 通常 2 个即可                              |
| `innodb_file_per_table`          | 建议开启，每张表独立 `.ibd` 文件，便于管理             |

`innodb_flush_log_at_trx_commit=2` 的含义是事务提交时写入 OS 缓冲，每秒刷盘一次。它不是绝对不丢数据，异常断电时可能丢最近 1 秒事务；但比每次提交都刷盘的性能压力小。

## MySQL 常用配置基线

```ini
[mysqld]
datadir=/data/mysql
max_connections=600
max_allowed_packet=32M
skip_name_resolve
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci

server-id=1
log-bin=/data/mysql-bin/binlog
binlog_format=ROW
sync_binlog=0
binlog_expire_logs_seconds=604800

slow_query_log=ON
long_query_time=5
slow_query_log_file=/data/mysql-slow/slow.log
log_queries_not_using_indexes=ON
log_throttle_queries_not_using_indexes=10

innodb_buffer_pool_size=4G
innodb_flush_log_at_trx_commit=2
innodb_log_buffer_size=16M
innodb_log_file_size=128M
innodb_log_files_in_group=2
innodb_file_per_table=1

[client]
default-character-set=utf8mb4
```

`datadir`、binlog 目录、慢查询目录都要提前创建并授权：

```bash
mkdir -p /data/mysql /data/mysql-bin /data/mysql-slow
chown -R mysql:mysql /data/mysql /data/mysql-bin /data/mysql-slow
```

## 主从复制原理

主从复制的核心流程是：主库把数据变更写入 binlog；从库 I/O 线程向主库请求 binlog；主库 log dump 线程发送 binlog；从库写入 relay log；从库 SQL 线程回放 relay log。

| 组件              | 位置 | 作用                        |
| --------------- | -- | ------------------------- |
| binlog          | 主库 | 记录数据变更                    |
| log dump thread | 主库 | 把 binlog 发送给从库            |
| I/O thread      | 从库 | 拉取主库 binlog 并写入 relay log |
| relay log       | 从库 | 中继日志                      |
| SQL thread      | 从库 | 回放 relay log              |

主从复制能用于读写分离、灾备、报表查询分流。但它不是强一致方案，复制延迟必须持续监控。

## 主库配置

```ini
[mysqld]
server-id=1
log-bin=/data/mysql-bin/binlog
binlog_format=ROW
```

创建复制用户：

```sql
CREATE USER 'rep'@'%' IDENTIFIED BY '复杂密码';
GRANT REPLICATION SLAVE ON *.* TO 'rep'@'%';
FLUSH PRIVILEGES;
```

导出需要同步的库，并记录复制位点：

```bash
mysqldump -uroot -p -B swk   --single-transaction   --master-data=2   > /backup/swk.sql
```

也可以查看当前主库位点：

```sql
SHOW MASTER STATUS\G
```

## 从库配置

```ini
[mysqld]
server-id=2
replicate-do-db=swk
```

导入主库备份：

```sql
SOURCE /backup/swk.sql;
```

MySQL 5.7 常见写法：

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

MySQL 8.0 之后推荐逐步使用 source/replica 术语：

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='192.168.31.10',
  SOURCE_PORT=3306,
  SOURCE_USER='rep',
  SOURCE_PASSWORD='复杂密码',
  SOURCE_LOG_FILE='binlog.000001',
  SOURCE_LOG_POS=154;

START REPLICA;
SHOW REPLICA STATUS\G
```

版本不同，命令支持情况不同。脚本里不要硬编码一种语法到所有环境。

## 主从状态检查

重点看三个字段：

| 字段                                                | 正常值   | 含义        |
| ------------------------------------------------- | ----- | --------- |
| `Slave_IO_Running` / `Replica_IO_Running`         | `Yes` | 是否能从主库拉日志 |
| `Slave_SQL_Running` / `Replica_SQL_Running`       | `Yes` | 是否能回放中继日志 |
| `Seconds_Behind_Master` / `Seconds_Behind_Source` | 接近 0  | 主从延迟      |

```sql
SHOW SLAVE STATUS\G
SHOW REPLICA STATUS\G
```

如果 SQL 线程停止，要优先看 `Last_SQL_Error`；如果 I/O 线程停止，要优先看网络、账号权限、密码、端口、防火墙、主库 binlog 文件是否仍存在。

## 主从启停顺序

```text
关闭：先关从库，再关主库
启动：先开主库，再开从库
```

这样可以减少从库找不到主库、主库位点变化不确定等问题。

## 主从延迟常见原因

| 原因      | 说明                            |
| ------- | ----------------------------- |
| 从库太多    | 多个从库直接拉主库 binlog，会增加主库网络和线程压力 |
| 从库硬件弱   | 从库磁盘、CPU、内存差于主库，回放速度跟不上       |
| 主库写入压力大 | binlog 产生速度超过从库回放速度           |
| 大事务     | 单个大事务会让复制延迟突然升高               |
| 慢 SQL   | 从库回放同样需要执行 SQL，慢修改会拖慢 SQL 线程  |
| 网络抖动    | I/O 线程断续拉取日志                  |

## 易错点

| 问题                                 | 处理                                         |
| ---------------------------------- | ------------------------------------------ |
| 慢查询日志无限增长                          | 用脚本切割，并执行 `flush-logs slow`                |
| 打开未使用索引记录后日志暴涨                     | 配 `log_throttle_queries_not_using_indexes` |
| `max_connections` 盲目调大             | 同时评估内存、线程、应用连接池                            |
| 从库 `server-id` 与主库重复               | 每台 MySQL 必须唯一                              |
| 从库只配置 `replicate-do-db` 却 SQL 写法混乱 | 注意 `USE db` 与跨库写入对过滤规则的影响                  |
| 主从状态只看一个 Yes                       | I/O 和 SQL 两个线程都要正常                         |

## 命令速查

```bash
mysqladmin -uroot -p flush-logs slow
mysqldumpslow -s t -t 10 /data/mysql-slow/slow.log
pt-query-digest /data/mysql-slow/slow.log > slow_report.txt
```

```sql
SHOW VARIABLES LIKE 'innodb%';
SHOW ENGINE INNODB STATUS\G
SHOW MASTER STATUS\G
SHOW SLAVE STATUS\G
SHOW REPLICA STATUS\G
```
