---
title: MySQL 约束、表关系、多表查询与事务管理
type: knowledge
status: evergreen
source: day20.md
created: 2026-06-14
updated: 2026-06-14
tags:
  - Linux
  - CentOS7
  - MySQL
  - SQL
  - 约束
  - 多表查询
  - 事务
  - 备份恢复
aliases:
  - DAY20
  - MySQL 高级
  - MySQL 约束
  - MySQL 多表查询
  - MySQL 事务
Belongs to:
  - "[[MySQL]]"
  - "[[数据库]]"
Related to:
  - "[[SQL 基础]]"
  - "[[DDL]]"
  - "[[DML]]"
  - "[[DQL]]"
  - "[[MySQL 用户权限]]"
  - "[[MySQL 备份恢复]]"
Has:
  - "[[非空约束]]"
  - "[[唯一约束]]"
  - "[[主键约束]]"
  - "[[默认约束]]"
  - "[[外键约束]]"
  - "[[一对多]]"
  - "[[多对多]]"
  - "[[一对一]]"
  - "[[内连接]]"
  - "[[外连接]]"
  - "[[子查询]]"
  - "[[事务]]"
  - "[[ACID]]"
  - "[[mysqldump]]"
  - "[[source 恢复]]"
  - "[[MySQL 用户管理]]"
---

# MySQL 约束、表关系、多表查询与事务管理

## 核心结论

MySQL 高级基础可以分成四条线：约束保证单表数据合法性，外键和表关系表达数据之间的关联，多表查询负责把关联数据查出来，事务保证一组写操作要么整体成功、要么整体失败。后面的用户权限、备份恢复、数据目录迁移，属于数据库运维的基础能力。

## 关系总览

| 条目 | 关系 |
|---|---|
| 约束 | 限制列或表中的数据规则 |
| 主键 | 一行数据的唯一标识 |
| 外键 | 建立两张表之间的数据库层面关系 |
| 一对多 | 在“多”的一方添加外键 |
| 多对多 | 使用中间表保存两边主键 |
| 一对一 | 外键加唯一约束 |
| 内连接 | 查询两表匹配的数据 |
| 外连接 | 保留主表未匹配数据 |
| 子查询 | 一个查询结果作为另一个查询的条件或临时表 |
| 事务 | 一组 SQL 的原子执行单元 |

## 约束

约束用于限制写入表的数据，保证数据正确性、有效性和完整性。

| 约束 | 关键字 | 作用 |
|---|---|---|
| 非空约束 | `NOT NULL` | 列值不能为 NULL |
| 唯一约束 | `UNIQUE` | 列值不能重复 |
| 主键约束 | `PRIMARY KEY` | 非空且唯一，一张表只能有一个主键 |
| 默认约束 | `DEFAULT` | 未指定值时使用默认值 |
| 外键约束 | `FOREIGN KEY` | 建立表关系 |
| 检查约束 | `CHECK` | 限制值范围；MySQL 8.0.16 后才真正执行 |

原始内容中提到 MySQL 不支持 CHECK，这在 MySQL 5.7 语境下基本成立。MySQL 8.0.16 之后 CHECK 才开始真正生效，所以整理时需要按版本区分。

## 非空约束

创建表时添加：

```sql
CREATE TABLE stu (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(20) NOT NULL
);
```

建表后添加：

```sql
ALTER TABLE stu MODIFY age INT NOT NULL;
```

删除非空约束：

```sql
ALTER TABLE stu MODIFY age INT;
```

## 唯一约束

创建表时添加：

```sql
CREATE TABLE user_account (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) UNIQUE
);
```

指定约束名：

```sql
CREATE TABLE user_account (
    id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50),
    CONSTRAINT uk_username UNIQUE(username)
);
```

建表后添加：

```sql
ALTER TABLE user_account ADD CONSTRAINT uk_username UNIQUE(username);
```

删除唯一约束：

```sql
ALTER TABLE user_account DROP INDEX uk_username;
```

唯一约束底层会创建唯一索引，所以删除时用 `DROP INDEX`。

## 主键约束

创建表时添加：

```sql
CREATE TABLE emp (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(20)
);
```

表级写法：

```sql
CREATE TABLE emp (
    id INT,
    name VARCHAR(20),
    CONSTRAINT pk_emp PRIMARY KEY(id)
);
```

建表后添加：

```sql
ALTER TABLE emp ADD PRIMARY KEY(id);
```

删除主键：

```sql
ALTER TABLE emp DROP PRIMARY KEY;
```

如果主键列有 `AUTO_INCREMENT`，删除主键前可能需要先修改字段属性。

## 默认约束

创建表时添加：

```sql
CREATE TABLE stu (
    id INT PRIMARY KEY AUTO_INCREMENT,
    sex CHAR(1) DEFAULT 'F'
);
```

建表后添加或修改：

```sql
ALTER TABLE stu MODIFY sex CHAR(1) DEFAULT 'F';
```

删除默认值：

```sql
ALTER TABLE stu MODIFY sex CHAR(1);
```

## 外键约束

外键用于让两张表建立数据库层面的关系。例如员工属于部门，员工表中的 `dep_id` 指向部门表中的 `id`。

创建表时添加：

```sql
CREATE TABLE dept (
    id INT PRIMARY KEY AUTO_INCREMENT,
    dep_name VARCHAR(20),
    addr VARCHAR(20)
);

CREATE TABLE emp (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(20),
    age INT,
    dep_id INT,
    CONSTRAINT fk_emp_dept FOREIGN KEY(dep_id) REFERENCES dept(id)
);
```

建表后添加：

```sql
ALTER TABLE emp
ADD CONSTRAINT fk_emp_dept
FOREIGN KEY(dep_id) REFERENCES dept(id);
```

删除外键：

```sql
ALTER TABLE emp DROP FOREIGN KEY fk_emp_dept;
```

外键可以保证引用一致性，但也会增加写入和删除时的约束检查。生产中是否使用数据库外键，要结合团队规范、业务复杂度和数据一致性要求决定。

## 员工表约束示例

```sql
CREATE TABLE emp (
    id INT PRIMARY KEY AUTO_INCREMENT,
    ename VARCHAR(50) NOT NULL UNIQUE,
    joindate DATE NOT NULL,
    salary DOUBLE(7,2) NOT NULL,
    bonus DOUBLE(7,2) DEFAULT 0
);
```

更适合金额的写法是使用 `DECIMAL`：

```sql
CREATE TABLE emp (
    id INT PRIMARY KEY AUTO_INCREMENT,
    ename VARCHAR(50) NOT NULL UNIQUE,
    joindate DATE NOT NULL,
    salary DECIMAL(10,2) NOT NULL,
    bonus DECIMAL(10,2) DEFAULT 0
);
```

## 数据库表关系

### 一对多

一个部门有多个员工，一个员工属于一个部门。实现方式是在多的一方添加外键。

```sql
CREATE TABLE tb_dept (
    id INT PRIMARY KEY AUTO_INCREMENT,
    dep_name VARCHAR(20),
    addr VARCHAR(20)
);

CREATE TABLE tb_emp (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(20),
    age INT,
    dep_id INT,
    CONSTRAINT fk_emp_dept FOREIGN KEY(dep_id) REFERENCES tb_dept(id)
);
```

### 多对多

一个订单包含多个商品，一个商品也可以属于多个订单。实现方式是建立中间表，中间表至少包含两个外键。

```sql
CREATE TABLE tb_order (
    id INT PRIMARY KEY AUTO_INCREMENT,
    payment DECIMAL(10,2),
    payment_type TINYINT,
    status TINYINT
);

CREATE TABLE tb_goods (
    id INT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(100),
    price DECIMAL(10,2)
);

CREATE TABLE tb_order_goods (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT,
    goods_id INT,
    count INT
);

ALTER TABLE tb_order_goods
ADD CONSTRAINT fk_order_id FOREIGN KEY(order_id) REFERENCES tb_order(id);

ALTER TABLE tb_order_goods
ADD CONSTRAINT fk_goods_id FOREIGN KEY(goods_id) REFERENCES tb_goods(id);
```

### 一对一

一对一常用于表拆分。常用字段放主表，不常用字段放详情表。实现方式是在任意一方加入外键，并加唯一约束。

```sql
CREATE TABLE tb_user_desc (
    id INT PRIMARY KEY AUTO_INCREMENT,
    city VARCHAR(20),
    edu VARCHAR(10),
    income INT,
    status CHAR(2),
    des VARCHAR(100)
);

CREATE TABLE tb_user (
    id INT PRIMARY KEY AUTO_INCREMENT,
    photo VARCHAR(100),
    nickname VARCHAR(50),
    age INT,
    gender CHAR(1),
    desc_id INT UNIQUE,
    CONSTRAINT fk_user_desc FOREIGN KEY(desc_id) REFERENCES tb_user_desc(id)
);
```

## 多表查询

多表查询本质是从多张表中取数据。没有连接条件时会产生笛卡尔积。

```sql
SELECT * FROM emp, dept;
```

加上关联条件：

```sql
SELECT *
FROM emp, dept
WHERE emp.dep_id = dept.did;
```

## 内连接

隐式内连接：

```sql
SELECT t1.name, t1.gender, t2.dname
FROM emp t1, dept t2
WHERE t1.dep_id = t2.did;
```

显式内连接：

```sql
SELECT *
FROM emp
INNER JOIN dept ON emp.dep_id = dept.did;
```

`INNER` 可以省略：

```sql
SELECT *
FROM emp
JOIN dept ON emp.dep_id = dept.did;
```

内连接只保留两张表能匹配上的数据。

## 外连接

左外连接：

```sql
SELECT *
FROM emp
LEFT JOIN dept ON emp.dep_id = dept.did;
```

右外连接：

```sql
SELECT *
FROM emp
RIGHT JOIN dept ON emp.dep_id = dept.did;
```

也可以交换表顺序，把右连接改成左连接：

```sql
SELECT *
FROM dept
LEFT JOIN emp ON emp.dep_id = dept.did;
```

外连接会保留主表中没有匹配到的数据，未匹配列显示为 NULL。

## 子查询

子查询是查询中嵌套查询。

单行单列子查询：

```sql
SELECT *
FROM emp
WHERE salary > (SELECT salary FROM emp WHERE name = '猪八戒');
```

多行单列子查询：

```sql
SELECT *
FROM emp
WHERE dep_id IN (
    SELECT did FROM dept WHERE dname = '财务部' OR dname = '市场部'
);
```

多行多列子查询作为临时表：

```sql
SELECT *
FROM (
    SELECT * FROM emp WHERE join_date > '2011-11-11'
) t1, dept
WHERE t1.dep_id = dept.did;
```

现代写法更推荐显式 JOIN：

```sql
SELECT *
FROM (
    SELECT * FROM emp WHERE join_date > '2011-11-11'
) t1
JOIN dept ON t1.dep_id = dept.did;
```

## 多表查询案例模式

查询员工编号、员工姓名、工资、职务名称、职务描述：

```sql
SELECT
    emp.id,
    emp.ename,
    emp.salary,
    job.jname,
    job.description
FROM emp
JOIN job ON emp.job_id = job.id;
```

查询员工、职务、部门信息：

```sql
SELECT
    emp.id,
    emp.ename,
    emp.salary,
    job.jname,
    job.description,
    dept.dname,
    dept.loc
FROM emp
JOIN job ON emp.job_id = job.id
JOIN dept ON dept.did = emp.dept_id;
```

查询工资等级：

```sql
SELECT
    emp.ename,
    emp.salary,
    salarygrade.grade
FROM emp
JOIN salarygrade
  ON emp.salary BETWEEN salarygrade.losalary AND salarygrade.hisalary;
```

查询部门人数：

```sql
SELECT
    dept.did,
    dept.dname,
    dept.loc,
    t1.emp_count
FROM dept
JOIN (
    SELECT dept_id, COUNT(*) AS emp_count
    FROM emp
    GROUP BY dept_id
) t1 ON dept.did = t1.dept_id;
```

如果想保留没有员工的部门，应使用左连接：

```sql
SELECT
    dept.did,
    dept.dname,
    dept.loc,
    COUNT(emp.id) AS emp_count
FROM dept
LEFT JOIN emp ON dept.did = emp.dept_id
GROUP BY dept.did, dept.dname, dept.loc;
```

## 事务

事务是一组数据库操作的整体提交或整体回滚机制。典型场景是转账：扣款和加款必须同时成功，否则就应回滚。

开启事务：

```sql
START TRANSACTION;
-- 或
BEGIN;
```

提交事务：

```sql
COMMIT;
```

回滚事务：

```sql
ROLLBACK;
```

转账示例：

```sql
BEGIN;

UPDATE account SET money = money - 500 WHERE name = '李四';
UPDATE account SET money = money + 500 WHERE name = '张三';

COMMIT;
```

如果中途失败：

```sql
ROLLBACK;
```

事务提交后不能再通过 `ROLLBACK` 撤销。

## ACID

| 特性 | 含义 |
|---|---|
| 原子性 Atomicity | 要么全部成功，要么全部失败 |
| 一致性 Consistency | 事务前后数据满足一致性约束 |
| 隔离性 Isolation | 多个事务之间相互影响受隔离级别控制 |
| 持久性 Durability | 提交后的修改持久保存 |

MySQL 默认自动提交：

```sql
SELECT @@autocommit;
```

关闭自动提交：

```sql
SET @@autocommit = 0;
```

开启自动提交：

```sql
SET @@autocommit = 1;
```

## 忘记 root 密码

处理流程：

```bash
systemctl stop mysqld
vim /etc/my.cnf
```

在 `[mysqld]` 下临时加入：

```ini
skip-grant-tables
```

启动：

```bash
systemctl start mysqld
mysql
```

修改密码：

```sql
UPDATE mysql.user
SET authentication_string = PASSWORD('新密码')
WHERE user = 'root' AND host = 'localhost';

FLUSH PRIVILEGES;
```

如果有远程 root：

```sql
UPDATE mysql.user
SET authentication_string = PASSWORD('新密码')
WHERE user = 'root' AND host = '%';

FLUSH PRIVILEGES;
```

最后删除 `skip-grant-tables` 并重启 MySQL：

```bash
systemctl restart mysqld
```

补充：MySQL 5.7 可用这种方式；MySQL 8.0 中认证字段和密码函数行为不同，更推荐 `ALTER USER`，并注意认证插件差异。

## 用户与权限管理

创建用户：

```sql
CREATE USER '用户名'@'主机' IDENTIFIED BY '密码';
```

授权：

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON xyj.* TO 'swk'@'%';
```

创建并授权：

```sql
GRANT ALL PRIVILEGES ON xyj.* TO 'zbj'@'%' IDENTIFIED BY '密码';
```

取消权限：

```sql
REVOKE SELECT ON xyj.* FROM 'zbj'@'%';
```

刷新权限：

```sql
FLUSH PRIVILEGES;
```

常用权限：

| 权限 | 作用 |
|---|---|
| `SELECT` | 查询 |
| `INSERT` | 插入 |
| `UPDATE` | 修改 |
| `DELETE` | 删除 |
| `CREATE` | 创建库表 |
| `DROP` | 删除库表 |
| `ALTER` | 修改表 |
| `INDEX` | 索引 |
| `EXECUTE` | 执行存储过程 |
| `REPLICATION SLAVE` | 主从复制 |
| `TRIGGER` | 触发器 |
| `ALL PRIVILEGES` | 所有权限 |

## MySQL 常用系统命令

```sql
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;
SHOW VARIABLES;
SHOW SESSION STATUS;
SHOW GLOBAL STATUS;
SHOW ENGINE INNODB STATUS;
SELECT NOW();
SELECT VERSION();
KILL 线程ID;
```

临时修改参数：

```sql
SET 参数 = 值;
```

重启后失效。永久修改应写入配置文件。

## 远程连接与非交互执行 SQL

远程连接：

```bash
mysql -h 192.168.31.23 -u root -p
```

指定端口：

```bash
mysql -h 192.168.31.23 -P 3306 -u root -p
```

不进入交互界面直接执行 SQL：

```bash
mysql -h'192.168.31.20' -u'zbj' -p'密码' -e "SELECT VERSION();"
```

输出到文件：

```bash
mysql -uroot -p'密码' -e "SHOW DATABASES;" > databases.txt
```

命令行直接写密码会暴露在历史记录或进程信息里，生产中更建议使用配置文件、环境变量或交互输入。

## mysqldump 备份

常用参数：

| 参数 | 作用 |
|---|---|
| `-B` | 备份多个库，并包含建库与 USE 语句 |
| `-d` | 只备份结构 |
| `-t` | 只备份数据 |

备份多个库：

```bash
mysqldump -uroot -p -B a b c > abc.sql
```

备份单库：

```bash
mysqldump -uroot -p -B xyj > xyj.sql
```

只备份结构：

```bash
mysqldump -uroot -p -d -B xyj > xyj_schema.sql
```

只备份数据：

```bash
mysqldump -uroot -p -t xyj > xyj_data.sql
```

原始内容中写到 `aa.sql.gz`，如果只是重定向到 `.gz` 文件名，并不会自动压缩。真正压缩应使用管道：

```bash
mysqldump -uroot -p -B xyj | gzip > xyj.sql.gz
```

恢复压缩备份：

```bash
gunzip -c xyj.sql.gz | mysql -uroot -p
```

## source 恢复

登录 MySQL 后恢复：

```sql
SOURCE /test/xyj.sql;
```

如果备份时使用了 `-B`，SQL 文件中有建库和 `USE` 语句，可以直接 source。没有使用 `-B` 时，需要先创建并选择数据库：

```sql
CREATE DATABASE xyj DEFAULT CHARACTER SET utf8mb4;
USE xyj;
SOURCE /test/xyj.sql;
```

命令行恢复：

```bash
mysql -uroot -p < /test/xyj.sql
```

## 数据目录迁移

新装时建议把 MySQL 数据目录放在单独分区或目录中，便于容量管理和备份。

新装前配置：

```ini
[mysqld]
datadir=/data/mysql
```

创建目录并授权：

```bash
mkdir -p /data
chown -R mysql:mysql /data
systemctl start mysqld
```

迁移已存在的数据目录：

```bash
systemctl stop mysqld
mkdir -p /data/mysql
chown -R mysql:mysql /data

cd /var/lib/mysql
cp -a * /data/mysql

vim /etc/my.cnf
```

修改：

```ini
[mysqld]
datadir=/data/mysql
```

确认权限：

```bash
chown -R mysql:mysql /data/mysql
systemctl start mysqld
```

如果 SELinux 开启，还要处理上下文，否则可能启动失败。

## 命令速查

| 目的 | 命令 |
|---|---|
| 查看线程 | `SHOW FULL PROCESSLIST;` |
| 查看变量 | `SHOW VARIABLES;` |
| 查看状态 | `SHOW GLOBAL STATUS;` |
| 查看 InnoDB | `SHOW ENGINE INNODB STATUS;` |
| 杀线程 | `KILL ID;` |
| 远程连接 | `mysql -h IP -u USER -p` |
| 执行 SQL | `mysql -uroot -p -e "SQL"` |
| 备份库 | `mysqldump -uroot -p -B db > db.sql` |
| 压缩备份 | `mysqldump -uroot -p -B db | gzip > db.sql.gz` |
| 恢复 | `mysql -uroot -p < db.sql` |

## 易错点

| 问题 | 处理 |
|---|---|
| 外键删除失败 | 先删除子表数据或删除外键 |
| `COUNT(列)` 少算 | NULL 不参与统计，统计行数用 `COUNT(*)` |
| 多表查询无条件 | 会产生笛卡尔积 |
| 外连接结果有 NULL | 主表无匹配数据，属于正常结果 |
| 事务提交后想回滚 | `COMMIT` 后不能 `ROLLBACK` |
| 备份文件后缀 `.gz` 但没压缩 | 必须经过 `gzip` |
| root 远程开放 `%` | 生产中限制来源 IP |
| 改 datadir 后启动失败 | 检查权限、SELinux、路径、配置 |

## 关联条目

| 类型 | 条目 |
|---|---|
| 前置 | [[SQL 基础]]、[[DDL]]、[[DML]]、[[DQL]] |
| 核心 | [[MySQL 约束]]、[[多表查询]]、[[MySQL 事务]]、[[MySQL 用户权限]] |
| 延伸 | [[MySQL 备份恢复]]、[[MySQL 主从复制]]、[[InnoDB]]、[[索引优化]] |
