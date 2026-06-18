---
type: Reference
status: Organized
Belongs to: "[[MySQL]]"
Related to:
  - "[[mysql-查询基础]]"
Has:
  - "[[外键约束]]"
  - "[[一对多]]"
  - "[[多对多]]"
  - "[[内连接]]"
  - "[[外连接]]"
  - "[[子查询]]"
source: day20.md
created: 2026-06-14
updated: 2026-06-14
---

# MySQL 约束、表关系与多表查询

## 约束

| 约束 | 关键字 | 作用 |
| --- | --- | --- |
| 非空约束 | `NOT NULL` | 列值不能为 NULL |
| 唯一约束 | `UNIQUE` | 列值不能重复 |
| 主键约束 | `PRIMARY KEY` | 非空且唯一 |
| 默认约束 | `DEFAULT` | 未指定值时使用默认值 |
| 外键约束 | `FOREIGN KEY` | 建立表关系 |

### 非空约束

```sql
CREATE TABLE stu (id INT PRIMARY KEY AUTO_INCREMENT, name VARCHAR(20) NOT NULL);
ALTER TABLE stu MODIFY age INT NOT NULL;
ALTER TABLE stu MODIFY age INT;       -- 删除非空
```

### 唯一约束

```sql
CREATE TABLE user_account (username VARCHAR(50) UNIQUE);
ALTER TABLE user_account ADD CONSTRAINT uk_username UNIQUE(username);
ALTER TABLE user_account DROP INDEX uk_username;
```

唯一约束底层创建唯一索引，用 `DROP INDEX` 删除。

### 主键约束

```sql
CREATE TABLE emp (id INT PRIMARY KEY AUTO_INCREMENT, name VARCHAR(20));
ALTER TABLE emp ADD PRIMARY KEY(id);
ALTER TABLE emp DROP PRIMARY KEY;
```

### 默认约束

```sql
CREATE TABLE stu (sex CHAR(1) DEFAULT 'F');
ALTER TABLE stu MODIFY sex CHAR(1) DEFAULT 'F';
ALTER TABLE stu MODIFY sex CHAR(1);   -- 删除默认值
```

### 外键约束

```sql
CREATE TABLE dept (id INT PRIMARY KEY AUTO_INCREMENT, dep_name VARCHAR(20));
CREATE TABLE emp (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(20),
    dep_id INT,
    CONSTRAINT fk_emp_dept FOREIGN KEY(dep_id) REFERENCES dept(id)
);

ALTER TABLE emp ADD CONSTRAINT fk_emp_dept FOREIGN KEY(dep_id) REFERENCES dept(id);
ALTER TABLE emp DROP FOREIGN KEY fk_emp_dept;
```

外键保证引用一致性，但增加约束检查成本。

## 表关系

### 一对多

在"多"的一方添加外键。

```sql
CREATE TABLE tb_dept (id INT PRIMARY KEY AUTO_INCREMENT, dep_name VARCHAR(20));
CREATE TABLE tb_emp (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(20),
    dep_id INT,
    CONSTRAINT fk_emp_dept FOREIGN KEY(dep_id) REFERENCES tb_dept(id)
);
```

### 多对多

建立中间表，包含两个外键。

```sql
CREATE TABLE tb_order (id INT PRIMARY KEY AUTO_INCREMENT, payment DECIMAL(10,2));
CREATE TABLE tb_goods (id INT PRIMARY KEY AUTO_INCREMENT, title VARCHAR(100));
CREATE TABLE tb_order_goods (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT, goods_id INT, count INT
);
ALTER TABLE tb_order_goods ADD CONSTRAINT fk_order FOREIGN KEY(order_id) REFERENCES tb_order(id);
ALTER TABLE tb_order_goods ADD CONSTRAINT fk_goods FOREIGN KEY(goods_id) REFERENCES tb_goods(id);
```

### 一对一

任意一方加入外键并加唯一约束。

## 多表查询

没有连接条件时产生笛卡尔积：

```sql
SELECT * FROM emp, dept WHERE emp.dep_id = dept.did;
```

### 内连接

只保留匹配的数据：

```sql
SELECT t1.name, t1.gender, t2.dname
FROM emp t1, dept t2
WHERE t1.dep_id = t2.did;

SELECT * FROM emp JOIN dept ON emp.dep_id = dept.did;
```

### 外连接

保留主表未匹配数据（未匹配列显示 NULL）：

```sql
SELECT * FROM emp LEFT JOIN dept ON emp.dep_id = dept.did;
SELECT * FROM dept LEFT JOIN emp ON emp.dep_id = dept.did;
```

### 子查询

```sql
SELECT * FROM emp WHERE salary > (SELECT salary FROM emp WHERE name = '猪八戒');

SELECT * FROM emp WHERE dep_id IN (
    SELECT did FROM dept WHERE dname = '财务部' OR dname = '市场部'
);

SELECT * FROM (SELECT * FROM emp WHERE join_date > '2011-11-11') t1
JOIN dept ON t1.dep_id = dept.did;
```

### 查询案例

查询部门人数，保留无员工部门：

```sql
SELECT dept.did, dept.dname, COUNT(emp.id) AS emp_count
FROM dept LEFT JOIN emp ON dept.did = emp.dept_id
GROUP BY dept.did, dept.dname, dept.loc;
```

## 易错点

| 问题 | 处理 |
| --- | --- |
| 外键删除失败 | 先删除子表数据或删除外键 |
| 多表查询无条件 | 会产生笛卡尔积 |
| 外连接结果有 NULL | 主表无匹配数据，属于正常结果 |
