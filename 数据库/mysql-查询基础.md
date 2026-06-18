---
type: Reference
status: Organized
Belongs to: "[[MySQL]]"
Related to:
  - "[[mysql-数据类型与-ddl]]"
Has:
  - "[[DML]]"
  - "[[DQL]]"
source: day19.md
created: 2026-06-14
updated: 2026-06-14
---

# MySQL 查询基础

## DML：插入、修改、删除

```sql
INSERT INTO stu (id, name) VALUES (1, '张三');

INSERT INTO stu VALUES (2, '李四', '男', '1999-11-11', 88.88, 'lisi@example.com', 1);

INSERT INTO stu (name, gender) VALUES
('李四1', '男'),
('李四2', '男');

UPDATE stu SET gender = '女' WHERE name = '张三';
UPDATE stu SET birthday = '1999-12-12', score = 99.99 WHERE name = '张三';

DELETE FROM stu WHERE name = '张三';
TRUNCATE TABLE stu;
```

`DELETE` 按行删除，可带条件；`TRUNCATE` 快速清空整张表。

## DQL：完整查询结构

```sql
SELECT 字段列表
FROM 表名列表
WHERE 条件列表
GROUP BY 分组字段
HAVING 分组后条件
ORDER BY 排序字段
LIMIT 分页限定;
```

执行顺序：`FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY -> LIMIT`

### 基础查询

```sql
SELECT name, age FROM stu;
SELECT DISTINCT address FROM stu;
SELECT name, math AS 数学成绩 FROM stu;
```

### 条件查询

```sql
SELECT * FROM stu WHERE age > 20;
SELECT * FROM stu WHERE age BETWEEN 20 AND 30;
SELECT * FROM stu WHERE age IN (18, 20, 22);
SELECT * FROM stu WHERE english IS NULL;
SELECT * FROM stu WHERE english IS NOT NULL;
```

NULL 不能用 `=` 判断。

### 模糊查询

| 通配符 | 含义 |
| --- | --- |
| `_` | 单个任意字符 |
| `%` | 任意多个字符 |

```sql
SELECT * FROM stu WHERE name LIKE '马%';
SELECT * FROM stu WHERE name LIKE '_花%';
```

### 排序查询

```sql
SELECT * FROM stu ORDER BY age ASC;
SELECT * FROM stu ORDER BY math DESC, english ASC;
```

多个排序字段时，前一个值相同后一个才起作用。

### 聚合函数

| 函数 | 作用 |
| --- | --- |
| `COUNT()` | 统计数量 |
| `MAX()` | 最大值 |
| `MIN()` | 最小值 |
| `SUM()` | 求和 |
| `AVG()` | 平均值 |

聚合函数忽略 NULL。

### 分组查询

```sql
SELECT sex, AVG(math), COUNT(*) FROM stu GROUP BY sex;
SELECT sex, AVG(math), COUNT(*)
FROM stu
WHERE math > 70
GROUP BY sex
HAVING COUNT(*) > 2;
```

`WHERE` 分组前过滤，`HAVING` 分组后过滤。

### 分页查询

```sql
SELECT * FROM stu LIMIT 0, 3;
SELECT * FROM stu LIMIT 40, 20;   -- 第 3 页，每页 20 条
```

大偏移分页可改用游标方式：

```sql
SELECT * FROM stu WHERE id > 10000 ORDER BY id LIMIT 20;
```

## 易错点

| 问题 | 处理 |
| --- | --- |
| `UPDATE` 忘记 WHERE | 会修改全表 |
| `DELETE` 忘记 WHERE | 会删除全表数据 |
| `NULL` 用 `=` 判断 | 必须使用 `IS NULL` |
| `GROUP BY` 查询非分组字段 | 严格模式下可能报错 |
| `COUNT(列)` 少算 | NULL 不参与统计，用 `COUNT(*)` |
