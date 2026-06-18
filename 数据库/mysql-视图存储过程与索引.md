---
type: Reference
status: Organized
Belongs to: "[[MySQL]]"
Related to:
  - "[[mysql-查询基础]]"
Has:
  - "[[MySQL 视图]]"
  - "[[存储过程]]"
  - "[[触发器]]"
  - "[[MySQL 索引]]"
source:
  - day21.md
created: 2026-06-14
updated: 2026-06-14
---

# MySQL 视图、存储过程与索引

## 视图

视图是虚拟表，保存查询逻辑，不保存查询结果。

```sql
CREATE OR REPLACE VIEW student_v AS
SELECT id, name FROM student WHERE id <= 20;

SHOW CREATE VIEW student_v\G
SELECT * FROM student_v;
ALTER VIEW student_v AS SELECT id, name FROM student WHERE id <= 30;
DROP VIEW IF EXISTS student_v;
```

### WITH CHECK OPTION

约束通过视图写入的数据必须满足视图筛选条件：

```sql
CREATE OR REPLACE VIEW student_v1 AS
SELECT id, name FROM student WHERE id <= 20
WITH CASCADED CHECK OPTION;
```

### 可更新视图限制

包含聚合、`DISTINCT`、`GROUP BY`、`UNION`、复杂表达式的视图通常不可更新。

## 存储过程

```sql
DELIMITER $$
CREATE PROCEDURE p_count()
BEGIN
  SELECT COUNT(*) FROM student;
END$$
DELIMITER ;

CALL p_count();
```

### 参数类型

| 参数 | 含义 |
| --- | --- |
| `IN` | 输入参数 |
| `OUT` | 输出参数 |
| `INOUT` | 输入输出参数 |

### 变量

| 类型 | 写法 | 作用域 |
| --- | --- | --- |
| 系统变量 | `@@global.max_connections` | 全局或会话 |
| 用户变量 | `@result` | 当前连接 |
| 局部变量 | `DECLARE total INT DEFAULT 0;` | BEGIN...END 块 |

### 游标与条件处理

```sql
DECLARE done INT DEFAULT 0;
DECLARE u_cursor CURSOR FOR SELECT ename, id FROM emp;
DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;
```

## 存储函数

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

## 触发器

```sql
CREATE TRIGGER account_insert_trigger
AFTER INSERT ON account
FOR EACH ROW
BEGIN
  INSERT INTO user_logs(operation, operate_time, operate_id)
  VALUES('insert', NOW(), NEW.id);
END;
```

| 操作 | 可用伪记录 |
| --- | --- |
| `INSERT` | `NEW` |
| `DELETE` | `OLD` |
| `UPDATE` | `OLD`、`NEW` |

## 索引

```sql
ALTER TABLE table_name ADD INDEX idx_col (col);
CREATE INDEX idx_col ON table_name(col);
DROP INDEX idx_col ON table_name;
```

适合建索引的场景：经常出现在 `WHERE`、`JOIN`、范围查询、`ORDER BY`、`GROUP BY` 中的列。

排查：

```sql
EXPLAIN SELECT * FROM emp WHERE name = 'Tom';
EXPLAIN ANALYZE SELECT * FROM emp WHERE name = 'Tom';  -- MySQL 8.0
SHOW INDEX FROM emp;
```

## 易错点

| 问题 | 处理 |
| --- | --- |
| 游标声明顺序错误 | 先变量，再 cursor，再 handler |
| 触发器里混用 `OLD`/`NEW` | INSERT 只有 NEW，DELETE 只有 OLD |
| 索引不是建了就一定用 | 需通过 EXPLAIN 验证执行计划 |
