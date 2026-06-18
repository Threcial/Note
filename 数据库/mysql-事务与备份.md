---
type: Reference
status: Organized
Belongs to: "[[MySQL]]"
Related to:
  - "[[mysql-约束表关系与多表查询]]"
Has:
  - "[[事务]]"
  - "[[ACID]]"
  - "[[mysqldump]]"
  - "[[MySQL 用户权限]]"
source:
  - day20.md
created: 2026-06-14
updated: 2026-06-14
---

# MySQL 事务与备份

## 事务

事务是一组 SQL 的原子执行单元：

```sql
BEGIN;
UPDATE account SET money = money - 500 WHERE name = '李四';
UPDATE account SET money = money + 500 WHERE name = '张三';
COMMIT;
```

中途失败时回滚：

```sql
ROLLBACK;
```

### ACID

| 特性 | 含义 |
| --- | --- |
| 原子性 | 要么全部成功，要么全部失败 |
| 一致性 | 事务前后数据满足约束 |
| 隔离性 | 多个事务之间受隔离级别控制 |
| 持久性 | 提交后的修改持久保存 |

MySQL 默认自动提交：

```sql
SELECT @@autocommit;
SET @@autocommit = 0;   -- 关闭自动提交
```

## 用户与权限管理

```sql
CREATE USER '用户名'@'主机' IDENTIFIED BY '密码';
GRANT SELECT, INSERT, UPDATE, DELETE ON xyj.* TO 'swk'@'%';
REVOKE SELECT ON xyj.* FROM 'zbj'@'%';
FLUSH PRIVILEGES;
```

常用权限：`SELECT`、`INSERT`、`UPDATE`、`DELETE`、`CREATE`、`DROP`、`ALTER`、`REPLICATION SLAVE`。

## mysqldump 备份

| 参数 | 作用 |
| --- | --- |
| `-B` | 备份多个库，包含建库和 USE 语句 |
| `-d` | 只备份结构 |
| `-t` | 只备份数据 |

```bash
mysqldump -uroot -p -B xyj > xyj.sql
mysqldump -uroot -p -d -B xyj > schema.sql
mysqldump -uroot -p -B xyj | gzip > xyj.sql.gz
```

## 恢复

```sql
SOURCE /test/xyj.sql;
```

```bash
mysql -uroot -p < /test/xyj.sql
gunzip -c xyj.sql.gz | mysql -uroot -p
```

## 忘记 root 密码

```bash
systemctl stop mysqld
```

在 `[mysqld]` 下添加 `skip-grant-tables`，启动后：

```sql
UPDATE mysql.user SET authentication_string = PASSWORD('新密码')
WHERE user = 'root' AND host = 'localhost';
FLUSH PRIVILEGES;
```

最后删除 `skip-grant-tables` 并重启。

## 常用系统命令

```sql
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;
SHOW VARIABLES;
SHOW GLOBAL STATUS;
SHOW ENGINE INNODB STATUS;
SELECT NOW();
SELECT VERSION();
KILL 线程ID;
```

## 易错点

| 问题 | 处理 |
| --- | --- |
| 事务提交后想回滚 | `COMMIT` 后不能 `ROLLBACK` |
| 备份文件后缀 `.gz` 但没压缩 | 必须经过 `gzip` |
| root 远程开放 `%` | 生产中限制来源 IP |
