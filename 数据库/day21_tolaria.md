---
title: MySQL 视图、存储程序、触发器、索引与 binlog 恢复
type: Reference
status: Organized
source:
  - day21.md
  - day21_视图、存储过程、触发器、索引.pdf
created: 2026-06-14
updated: 2026-06-14
Belongs to: "[[MySQL]]"
Related to:
  - "[[MySQL 主从复制]]"
Has:
  - "[[MySQL 视图]]"
  - "[[存储过程]]"
  - "[[触发器]]"
  - "[[MySQL 索引]]"
  - "[[binlog]]"
---
# MySQL 视图、存储程序、触发器、索引与 binlog 恢复

这一组内容开始从“会写 SQL”转到“数据库对象和恢复链路”。视图、存储过程、函数、触发器、索引都属于数据库内部能力；binlog、mysqldump、mysqlbinlog 则是运维侧恢复链路的核心

## 关系整理

| 主题     | 位置      | 作用                                  |
| ------ | ------- | ----------------------------------- |
| 视图     | 查询层     | 封装查询逻辑，隐藏底层表结构，简化访问                 |
| 存储过程   | 数据库执行层  | 把多条 SQL 和流程控制封装成可调用程序               |
| 存储函数   | 数据库表达式层 | 带返回值，可被 `SELECT` 调用                 |
| 游标     | 存储程序内部  | 逐行读取结果集，用于循环处理                      |
| 条件处理程序 | 存储程序内部  | 捕获 `NOT FOUND`、异常和警告，控制流程继续或退出      |
| 触发器    | 表事件层    | 在 `INSERT`、`UPDATE`、`DELETE` 前后自动执行 |
| 索引     | 存储引擎层   | 减少扫描数据量，提高检索、排序、连接效率                |
| binlog | 日志恢复层   | 记录数据变更，用于主从复制和增量恢复                  |

## 视图

视图是虚拟表。它保存的是查询逻辑，不保存查询结果；访问视图时，MySQL 会根据视图定义的查询语句动态生成结果。

视图适合三个场景：第一，隐藏复杂查询，把多表连接或过滤条件封装成一个名字；第二，限制用户只能访问部分列或部分行；第三，在底层表结构变化时，通过调整视图降低对上层查询的影响。

### 创建、查看、修改、删除

```sql
CREATE OR REPLACE VIEW student_v AS
SELECT id, name
FROM student
WHERE id <= 20;

SHOW CREATE VIEW student_v\G
SELECT * FROM student_v;

ALTER VIEW student_v AS
SELECT id, name
FROM student
WHERE id <= 30;

DROP VIEW IF EXISTS student_v;
```

`CREATE OR REPLACE VIEW` 适合反复调整视图定义。`SHOW CREATE VIEW` 可以看到视图原始定义，排查视图权限、检查选项和字段名时很有用。

### WITH CHECK OPTION

`WITH CHECK OPTION` 约束通过视图写入的数据必须满足视图自己的筛选条件。不加检查选项时，可能通过视图插入一条“插完以后自己在视图里看不到”的数据。

```sql
CREATE OR REPLACE VIEW student_v1 AS
SELECT id, name
FROM student
WHERE id <= 20
WITH CASCADED CHECK OPTION;

INSERT INTO student_v1 VALUES (5, 'tom');
INSERT INTO student_v1 VALUES (25, 'tom'); -- 不满足 id <= 20，会被阻止
```

`CASCADED` 是默认行为，会递归检查依赖视图链上的条件。`LOCAL` 只要求当前视图以及显式带检查选项的下层视图生效，不会自动把检查选项向下级视图传播。

### 可更新视图的限制

视图能不能更新，关键看视图结果和基表行之间是否还能保持一对一关系。包含以下内容时，通常不可更新：

| 内容                    | 影响               |
| --------------------- | ---------------- |
| 聚合函数                  | 一行结果可能来自多行基表     |
| `DISTINCT`            | 去重后无法定位原始行       |
| `GROUP BY` / `HAVING` | 结果是分组后的统计值       |
| `UNION` / `UNION ALL` | 行来源可能来自多个查询分支    |
| 复杂表达式列                | 修改表达式结果无法反推回基表字段 |

## 存储过程

存储过程是保存在数据库中的 SQL 代码块。它可以包含变量、判断、循环、游标和异常处理，适合把靠近数据的批处理逻辑封装起来。

```sql
DELIMITER $$
CREATE PROCEDURE p_count()
BEGIN
  SELECT COUNT(*) FROM student;
END$$
DELIMITER ;

CALL p_count();
```

在命令行里创建存储过程时需要临时修改分隔符。否则过程体内部的分号会被客户端提前识别为语句结束。

### 参数类型

| 参数      | 含义     | 典型用法            |
| ------- | ------ | --------------- |
| `IN`    | 输入参数   | 调用时传入查询条件、数量、阈值 |
| `OUT`   | 输出参数   | 过程执行后把结果写入变量    |
| `INOUT` | 输入输出参数 | 传入初始值，过程内修改后返回  |

```sql
DELIMITER $$
CREATE PROCEDURE p_score(IN score INT, OUT result VARCHAR(10))
BEGIN
  IF score >= 85 THEN
    SET result = '优秀';
  ELSEIF score >= 60 THEN
    SET result = '及格';
  ELSE
    SET result = '不及格';
  END IF;
END$$
DELIMITER ;

CALL p_score(58, @result);
SELECT @result;
```

### 变量

| 类型   | 写法                             | 作用域                  |
| ---- | ------------------------------ | -------------------- |
| 系统变量 | `@@global.max_connections`     | 全局或当前会话              |
| 用户变量 | `@result`                      | 当前连接                 |
| 局部变量 | `DECLARE total INT DEFAULT 0;` | 当前 `BEGIN ... END` 块 |

```sql
SHOW GLOBAL VARIABLES LIKE 'max_connections';
SELECT @@session.autocommit;

SET @num := 0;
SELECT COUNT(*) INTO @num FROM student;
SELECT @num;
```

## 游标和条件处理程序

游标用于逐行读取查询结果。MySQL 的游标只能在存储过程或函数中使用，声明顺序必须注意：先声明变量，再声明游标，再声明 handler。如果把游标声明放在建表语句之后，会直接报错。

错误思路是用 `WHILE TRUE` 死循环不断 `FETCH`。当游标已经没有数据时，会触发 `NOT FOUND`，如果没有 handler，就会报错。

更稳的写法是定义 `done` 标志位，并通过 `DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;` 标记游标结束。

```sql
DELIMITER $$
CREATE PROCEDURE p_cursor(IN uage INT)
BEGIN
  DECLARE uname VARCHAR(100);
  DECLARE uid INT;
  DECLARE done INT DEFAULT 0;

  DECLARE u_cursor CURSOR FOR
    SELECT ename, id FROM emp WHERE id <= uage;

  DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;

  DROP TABLE IF EXISTS tb_user_pro;
  CREATE TABLE IF NOT EXISTS tb_user_pro(
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    uid INT
  );

  OPEN u_cursor;

  read_loop: LOOP
    FETCH u_cursor INTO uname, uid;
    IF done = 1 THEN
      LEAVE read_loop;
    END IF;
    INSERT INTO tb_user_pro(name, uid) VALUES (uname, uid);
  END LOOP;

  CLOSE u_cursor;
END$$
DELIMITER ;
```

这里的 `LOOP + LEAVE` 比 `WHILE done <= 0` 更直观：每次 `FETCH` 后立刻判断是否结束，结束就跳出循环，不会把最后一次空结果插入目标表。

## 存储函数

存储函数和存储过程的区别是：函数必须有返回值，并且可以在 `SELECT` 中调用。函数参数只能是输入参数。

```sql
DELIMITER $$
CREATE FUNCTION fun_sum(n INT)
RETURNS INT DETERMINISTIC
BEGIN
  DECLARE total INT DEFAULT 0;
  WHILE n > 0 DO
    SET total = total + n;
    SET n = n - 1;
  END WHILE;
  RETURN total;
END$$
DELIMITER ;

SELECT fun_sum(100);
```

`DETERMINISTIC` 表示相同输入一定产生相同输出。MySQL 对函数的确定性、安全性和二进制日志有额外检查，生产环境里函数不要写有副作用的复杂逻辑。

## 触发器

触发器绑定在表事件上，在 `INSERT`、`UPDATE`、`DELETE` 之前或之后自动执行。MySQL 触发器是行级触发器，`FOR EACH ROW` 表示每影响一行都会执行一次。

| 操作       | 可用伪记录       | 含义         |
| -------- | ----------- | ---------- |
| `INSERT` | `NEW`       | 新插入的数据     |
| `DELETE` | `OLD`       | 被删除的旧数据    |
| `UPDATE` | `OLD`、`NEW` | 更新前和更新后的数据 |

先建操作日志表：

```sql
CREATE TABLE user_logs(
  id INT PRIMARY KEY AUTO_INCREMENT,
  operation VARCHAR(20) NOT NULL COMMENT '操作类型 insert/update/delete',
  operate_time DATETIME NOT NULL COMMENT '操作时间',
  operate_id INT NOT NULL COMMENT '操作的ID',
  operate_params VARCHAR(500) COMMENT '操作参数'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

插入触发器：

```sql
DELIMITER $$
CREATE TRIGGER account_insert_trigger
AFTER INSERT ON account
FOR EACH ROW
BEGIN
  INSERT INTO user_logs(operation, operate_time, operate_id, operate_params)
  VALUES('insert', NOW(), NEW.id,
         CONCAT('插入的数据内容为:id=', NEW.id, ', name=', NEW.name));
END$$
DELIMITER ;
```

删除触发器：

```sql
DELIMITER $$
CREATE TRIGGER account_delete_trigger
AFTER DELETE ON account
FOR EACH ROW
BEGIN
  INSERT INTO user_logs(operation, operate_time, operate_id, operate_params)
  VALUES('delete', NOW(), OLD.id,
         CONCAT('删除的数据内容为:id=', OLD.id, ', name=', OLD.name));
END$$
DELIMITER ;
```

更新触发器：

```sql
DELIMITER $$
CREATE TRIGGER account_update_trigger
AFTER UPDATE ON account
FOR EACH ROW
BEGIN
  INSERT INTO user_logs(operation, operate_time, operate_id, operate_params)
  VALUES('update', NOW(), NEW.id,
         CONCAT('更新的数据内容为:id=', OLD.id, ' -> ', NEW.id,
                ', name=', OLD.name, ' -> ', NEW.name));
END$$
DELIMITER ;
```

触发器适合做简单审计、数据校验和冗余字段维护。复杂业务逻辑不适合大量放进触发器，因为触发器是隐式执行，排查链路比显式 SQL 更困难。

## 索引

索引用于减少扫描数据量。InnoDB 表本身按主键组织，主键索引也叫聚簇索引；普通索引、唯一索引属于二级索引，二级索引叶子节点保存主键值，再通过主键回表找到完整行。

### 创建与删除

```sql
ALTER TABLE table_name ADD INDEX idx_col (col);
ALTER TABLE table_name ADD UNIQUE uk_col (col);
ALTER TABLE table_name ADD PRIMARY KEY (id);

CREATE INDEX idx_col ON table_name(col);
CREATE UNIQUE INDEX uk_col ON table_name(col);

DROP INDEX idx_col ON table_name;
ALTER TABLE table_name DROP INDEX idx_col;
ALTER TABLE table_name DROP PRIMARY KEY;
```

`ALTER TABLE ... ADD INDEX` 和 `CREATE INDEX` 都能创建索引。实际维护表结构时，`ALTER TABLE` 更统一；单独补普通索引时，`CREATE INDEX` 更直观。

### 应该考虑建索引的列

| 场景                           | 原因              |
| ---------------------------- | --------------- |
| 经常出现在 `WHERE` 中              | 减少过滤扫描          |
| 经常用于 `JOIN`                  | 提高连接效率          |
| 经常用于范围查询                     | B+Tree 天然支持有序范围 |
| 经常用于 `ORDER BY` / `GROUP BY` | 可能减少排序和临时表      |
| 唯一性要求                        | 通过唯一索引保证数据不重复   |

不适合盲目建索引的情况：小表、低区分度字段、频繁写入但很少查询的字段、超长字符串全文索引需求。字符串字段可以考虑短索引，例如 `INDEX idx_name(name(20))`，减少索引体积。

### 现代排查补充

```sql
EXPLAIN SELECT * FROM emp WHERE name = 'Tom';
EXPLAIN ANALYZE SELECT * FROM emp WHERE name = 'Tom'; -- MySQL 8.0 可用
SHOW INDEX FROM emp;
```

`EXPLAIN` 看执行计划，`EXPLAIN ANALYZE` 可以看到实际执行耗时。判断索引是否有效不能只看“有没有建”，还要看执行计划是否使用、扫描行数是否下降、是否出现临时表和文件排序。

## binlog

binlog 是 MySQL 的二进制日志，记录数据变更，主要用于主从复制和增量恢复。查询语句不会写入 binlog，`INSERT`、`UPDATE`、`DELETE`、DDL 变更会写入。

### 开启 binlog

```ini
[mysqld]
server-id=1
log-bin=/var/lib/mysql/mysql-bin/binlog
binlog_format=ROW
sync_binlog=0
max_binlog_size=500M
expire_logs_days=7
```

目录必须存在，并且属主属组需要是 `mysql:mysql`。`server-id` 必须配置，否则 binlog 或复制相关配置可能无法按预期工作。

MySQL 8.0 更推荐使用 `binlog_expire_logs_seconds` 控制过期时间，例如：

```ini
binlog_expire_logs_seconds=604800
```

`sync_binlog` 控制 binlog 刷盘策略。`0` 性能最好，由操作系统决定刷盘；`1` 安全性最好，每次事务提交都刷盘；`N` 表示每 N 次事务提交刷盘一次。

### 查看 binlog

```sql
SHOW VARIABLES LIKE '%binlog%';
SHOW VARIABLES LIKE 'max_binlog_size';
SHOW MASTER STATUS;
```

MySQL 8.4 新版本中，部分主从相关术语逐步替换为 source/replica。写脚本时要注意版本差异，不能把所有环境都按 5.7 的输出处理。

### mysqlbinlog 常用参数

| 参数                 | 作用                         |
| ------------------ | -------------------------- |
| `--no-defaults`    | 避免客户端默认配置导致字符集等问题          |
| `-d dbname`        | 只解析指定数据库相关事件               |
| `-r file.sql`      | 输出到 SQL 文件                 |
| `--start-position` | 指定起始 position              |
| `--stop-position`  | 指定结束 position              |
| `--start-datetime` | 指定起始时间                     |
| `--stop-datetime`  | 指定结束时间                     |
| `--skip-gtids`     | 跳过 GTID 信息，恢复到非 GTID 环境时常用 |

```bash
mysqlbinlog --no-defaults binlog.000002 -r all.sql
mysqlbinlog --no-defaults -d swk --skip-gtids binlog.000002 -r swk.sql
mysqlbinlog --no-defaults binlog.000001 binlog.000002   -d swk --skip-gtids --start-position=2792 -r swk.sql
```

按 position 恢复更精确。按 datetime 恢复只能精确到秒，1 秒内有多条变更时不够细。

## 全量备份与增量恢复

全量备份通常来自 `mysqldump`，增量恢复依赖 binlog。

### 备份一致性

`mysqldump` 默认执行过程中数据库仍然可以写入，如果不处理一致性，备份文件可能不是同一时刻的数据快照。常用做法是加：

```bash
mysqldump -uroot -p -B swk   --single-transaction   --master-data=2   > swk_full.sql
```

`--single-transaction` 依赖 InnoDB 的事务一致性视图，适合 InnoDB 表；`--master-data=2` 会把 binlog 文件名和 position 以注释方式写入备份文件，便于后续增量恢复定位临界点。

MySQL 8.0/8.4 环境中，`--master-data` 在新文档语义中逐步被 `--source-data` 替代。跨版本写自动化脚本时，需要根据 `mysqldump --help` 输出确认当前版本支持的参数。

### 恢复流程

恢复前先阻断业务写入，只保留本机恢复操作，避免恢复过程中继续产生新数据。

```sql
-- 先恢复全量备份
SOURCE /backup/swk_full.sql;
```

根据全量备份中记录的 binlog 文件和 position，生成增量 SQL：

```bash
mysqlbinlog --no-defaults binlog.000016 binlog.000017   -d swk --skip-gtids --start-position=2725   -r swk_incr.sql
```

再恢复增量：

```sql
SOURCE /backup/swk_incr.sql;
```

恢复完成并校验数据后，再恢复 binlog 配置和业务连接。

## md5sum 校验

备份文件远程传输后需要校验是否损坏。`md5sum` 可以生成校验文件，再在目标端验证。

```bash
md5sum swk_full.sql > swk_full.sql.flag
md5sum -c swk_full.sql.flag
```

输出 `swk_full.sql: OK` 表示文件内容一致。脚本中可以通过 `$?` 判断校验结果，非 0 直接终止恢复或传输流程。

## 易错点

| 问题                       | 处理                                                |
| ------------------------ | ------------------------------------------------- |
| 游标死循环报 `NOT FOUND`       | 定义 `done` 标志和 `CONTINUE HANDLER`                  |
| 游标声明顺序错误                 | 先变量，再 cursor，再 handler                            |
| `WITH CHECK OPTION` 误解   | 它限制通过视图写入的数据，不只是查询过滤                              |
| 触发器里混用 `OLD`/`NEW`       | `INSERT` 只有 `NEW`，`DELETE` 只有 `OLD`，`UPDATE` 两者都有 |
| binlog 目录权限不对            | 目录提前创建并授权给 `mysql:mysql`                          |
| 只恢复全量不恢复 binlog          | 只能恢复到备份时间点                                        |
| 用 datetime 恢复要求精确到某条 SQL | 改用 position 更可靠                                   |
| 备份时没有记录 position         | 后续增量恢复边界会变得不确定                                    |

## 命令速查

```sql
SHOW CREATE VIEW view_name\G
SHOW TRIGGERS\G
SHOW PROCEDURE STATUS WHERE Db = DATABASE();
SHOW CREATE PROCEDURE proc_name\G
SHOW INDEX FROM table_name;
SHOW MASTER STATUS;
```

```bash
mysqlbinlog --no-defaults binlog.000001 -r recover.sql
md5sum file.sql > file.sql.flag
md5sum -c file.sql.flag
```
