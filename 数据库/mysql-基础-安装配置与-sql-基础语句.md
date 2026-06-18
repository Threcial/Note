---
type: Reference
status: Organized
source: day19.md
created: 2026-06-14
updated: 2026-06-14
Belongs to: "[[MySQL]]"
Related to:
  - "[[Linux 软件包管理]]"
  - "[[字符集]]"
Has:
  - "[[SQL]]"
  - "[[MySQL 字符集]]"
  - "[[MySQL 数据类型]]"
  - "[[DDL]]"
  - "[[DML]]"
  - "[[DQL]]"
---
# MySQL 基础、安装配置与 SQL 基础语句

数据库解决的是持久化、有组织存储和高效管理数据的问题。MySQL 是数据库管理系统，不是单纯的“数据库文件”。日常使用里需要同时理解三层关系：数据库服务进程负责管理数据，SQL 是和数据库交互的语言，具体的库、表、行、列是数据组织形式。

MySQL 基础阶段的重点是：安装与初始化、字符集、数据类型、DDL、DML、DQL。字符集建议统一使用 `utf8mb4`，不要再使用 MySQL 早期的 `utf8` 作为默认选择。

## 关系总览

| 条目        | 关系                    |
| --------- | --------------------- |
| DB        | 数据库，用于组织存储数据          |
| DBMS      | 数据库管理系统，MySQL 属于 DBMS |
| SQL       | 操作关系型数据库的语言           |
| DDL       | 定义库、表、字段              |
| DML       | 增删改表中数据               |
| DQL       | 查询数据                  |
| DCL       | 用户和权限控制               |
| `utf8mb4` | MySQL 中更完整的 UTF-8 字符集 |

## 数据库与 DBMS

文件也可以持久化数据，但当数据量变大、并发读写变多、需要条件查询和事务一致性时，普通文件维护成本会迅速上升。数据库管理系统的价值在于提供：

| 能力   | 说明           |
| ---- | ------------ |
| 数据组织 | 库、表、字段、索引    |
| 查询能力 | 通过 SQL 按条件查询 |
| 并发控制 | 多用户同时访问      |
| 事务   | 保证一组操作的一致性   |
| 权限   | 控制用户能访问哪些库表  |
| 恢复   | 备份、日志、恢复机制   |

常见 DBMS：

| 系统         | 特点           |
| ---------- | ------------ |
| MySQL      | 开源常用，中小型业务常见 |
| MariaDB    | MySQL 分支，开源  |
| PostgreSQL | 开源，功能强，标准支持好 |
| Oracle     | 商业大型数据库      |
| SQL Server | 微软生态常见       |
| SQLite     | 嵌入式轻量数据库     |

## SQL 分类

| 分类  | 全称                         | 作用         |
| --- | -------------------------- | ---------- |
| DDL | Data Definition Language   | 定义库、表、字段   |
| DML | Data Manipulation Language | 插入、修改、删除数据 |
| DQL | Data Query Language        | 查询数据       |
| DCL | Data Control Language      | 用户、权限、安全控制 |

SQL 可以单行或多行书写，以分号结尾。MySQL 关键字不区分大小写，但建议关键字大写，库表字段按团队规范统一。

注释：

```sql
-- 单行注释，注意 -- 后面需要空格
# MySQL 单行注释
/* 多行注释 */
```

命令行中使用 `\G` 可以纵向显示结果：

```sql
SELECT * FROM mysql.user\G
```

## MySQL 字符集

创建数据库时建议明确指定字符集和排序规则：

```sql
CREATE DATABASE app_db
  DEFAULT CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

常用选择：

| 项目      | 建议                   |
| ------- | -------------------- |
| 字符集     | `utf8mb4`            |
| 通用排序    | `utf8mb4_unicode_ci` |
| 更快但不够精确 | `utf8mb4_general_ci` |

查看字符集：

```sql
SHOW VARIABLES LIKE '%chara%';
```

查看排序规则：

```sql
SHOW VARIABLES LIKE '%coll%';
```

关键变量：

| 变量                         | 含义               |
| -------------------------- | ---------------- |
| `character_set_client`     | 客户端发送 SQL 使用的字符集 |
| `character_set_connection` | 连接层字符集           |
| `character_set_database`   | 当前数据库默认字符集       |
| `character_set_results`    | 结果集字符集           |
| `character_set_server`     | 服务端默认字符集         |
| `collation_server`         | 服务端默认排序规则        |

## MySQL 5.7 RPM 安装

卸载可能冲突的 MariaDB 库：

```bash
rpm -e --nodeps mariadb-libs
```

安装依赖：

```bash
yum install ncurses-devel libaio-devel -y
```

按顺序安装 RPM 包：

```bash
rpm -ivh mysql-community-common-5.7.38-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.38-1.el7.x86_64.rpm
rpm -ivh mysql-community-devel-5.7.38-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-5.7.38-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-5.7.38-1.el7.x86_64.rpm
```

配置文件：

```bash
vim /etc/my.cnf
```

推荐基础配置：

```ini
[mysqld]
max_connections=200
max_allowed_packet=32M
skip_name_resolve
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci

[client]
default-character-set=utf8mb4
```

参数说明：

| 参数                     | 作用                     |
| ---------------------- | ---------------------- |
| `max_connections`      | 最大连接数                  |
| `max_allowed_packet`   | 单次通信包大小，避免大 SQL 或大字段失败 |
| `skip_name_resolve`    | 跳过反向 DNS 解析，加快连接       |
| `character-set-server` | 服务端默认字符集               |
| `collation-server`     | 服务端默认排序规则              |

启动：

```bash
systemctl enable --now mysqld
```

查找初始密码：

```bash
grep 'temporary password' /var/log/mysqld.log
```

登录并修改密码：

```bash
mysql -uroot -p
```

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY '复杂密码';
FLUSH PRIVILEGES;
```

## root 远程登录

默认 root 通常只允许本机登录。远程 root 风险很高，正式环境更建议创建专用业务用户，并限制来源 IP。

创建允许任意主机连接的 root 用户：

```sql
CREATE USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '复杂密码';
GRANT ALL ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

更安全的写法是限制来源：

```sql
CREATE USER 'app_admin'@'192.168.31.%' IDENTIFIED BY '复杂密码';
GRANT ALL ON app_db.* TO 'app_admin'@'192.168.31.%';
FLUSH PRIVILEGES;
```

## MySQL 数据类型

### 整数

| 类型        | 字节 | 有符号范围                    | 无符号范围          |
| --------- | -- | ------------------------ | -------------- |
| `TINYINT` | 1  | -128 到 127               | 0 到 255        |
| `INT`     | 4  | -2147483648 到 2147483647 | 0 到 4294967295 |
| `BIGINT`  | 8  | 很大                       | 更大             |

### 小数

| 类型             | 特点            |
| -------------- | ------------- |
| `FLOAT`        | 近似值，不推荐用于精确计算 |
| `DOUBLE`       | 近似值，不推荐用于精确计算 |
| `DECIMAL(m,d)` | 精确小数，适合金额     |

`DECIMAL(8,2)` 表示总共 8 位，小数点后 2 位。小数位超出会按规则处理，例如 `DECIMAL(5,2)` 存 `2.009` 可能变成 `2.01`。

### 字符串

| 类型           | 特点              |
| ------------ | --------------- |
| `CHAR(n)`    | 固定长度，最大 255 字符  |
| `VARCHAR(n)` | 可变长度，常用         |
| `TEXT`       | 长文本，最大 65535 字符 |

### 时间

| 类型          | 特点                         |
| ----------- | -------------------------- |
| `DATE`      | 年月日                        |
| `DATETIME`  | 年月日时分秒，与时区无关               |
| `TIMESTAMP` | 存储时与时区转换相关，旧版本范围到 2038 年附近 |

## DDL：数据库操作

查看数据库：

```sql
SHOW DATABASES;
```

创建数据库：

```sql
CREATE DATABASE app_db;
CREATE DATABASE IF NOT EXISTS app_db;
```

删除数据库：

```sql
DROP DATABASE app_db;
DROP DATABASE IF EXISTS app_db;
```

使用数据库：

```sql
USE app_db;
```

查看当前数据库：

```sql
SELECT DATABASE();
```

## DDL：表操作

查看当前库的表：

```sql
SHOW TABLES;
```

查看表结构：

```sql
DESC 表名;
```

创建表：

```sql
CREATE TABLE student (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(10),
    gender CHAR(1),
    birthday DATE,
    score DECIMAL(5,2),
    email VARCHAR(64),
    tel VARCHAR(15),
    status TINYINT
);
```

主键：

```sql
id INT PRIMARY KEY AUTO_INCREMENT
```

主键要求非空且唯一，通常作为一行数据的唯一标识。

删除表：

```sql
DROP TABLE student;
DROP TABLE IF EXISTS student;
```

修改表名：

```sql
ALTER TABLE student RENAME TO stu;
```

添加列：

```sql
ALTER TABLE stu ADD address VARCHAR(50);
```

修改列类型：

```sql
ALTER TABLE stu MODIFY address CHAR(50);
```

修改列名和类型：

```sql
ALTER TABLE stu CHANGE address addr VARCHAR(50);
```

删除列：

```sql
ALTER TABLE stu DROP addr;
```

## DML：插入、修改、删除

指定列插入：

```sql
INSERT INTO stu (id, name) VALUES (1, '张三');
```

全部列插入：

```sql
INSERT INTO stu VALUES (2, '李四', '男', '1999-11-11', 88.88, 'lisi@example.com', '13888888888', 1);
```

批量插入：

```sql
INSERT INTO stu (name, gender) VALUES
('李四1', '男'),
('李四2', '男'),
('李四3', '男');
```

修改数据：

```sql
UPDATE stu SET gender = '女' WHERE name = '张三';
UPDATE stu SET birthday = '1999-12-12', score = 99.99 WHERE name = '张三';
```

删除数据：

```sql
DELETE FROM stu WHERE name = '张三';
```

清空表：

```sql
TRUNCATE TABLE stu;
```

`DELETE` 是按行删除，可以带条件；`TRUNCATE` 是快速清空整张表，通常不可按条件删除。

## DQL：查询语法

完整查询结构：

```sql
SELECT 字段列表
FROM 表名列表
WHERE 条件列表
GROUP BY 分组字段
HAVING 分组后条件
ORDER BY 排序字段
LIMIT 分页限定;
```

执行理解顺序通常是：

```text
FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY -> LIMIT
```

## 基础查询

查询指定字段：

```sql
SELECT name, age FROM stu;
```

查询全部字段：

```sql
SELECT * FROM stu;
```

去重：

```sql
SELECT DISTINCT address FROM stu;
```

别名：

```sql
SELECT name, math AS 数学成绩, english AS 英语成绩 FROM stu;
SELECT name, math 数学成绩, english 英语成绩 FROM stu;
```

生产中不建议长期使用 `SELECT *`，字段多时会增加网络传输，也容易受表结构变动影响。

## 条件查询

```sql
SELECT * FROM stu WHERE age > 20;
SELECT * FROM stu WHERE age >= 20 AND age <= 30;
SELECT * FROM stu WHERE age BETWEEN 20 AND 30;
SELECT * FROM stu WHERE age IN (18, 20, 22);
SELECT * FROM stu WHERE age NOT IN (55, 45, 57);
```

NULL 判断：

```sql
SELECT * FROM stu WHERE english IS NULL;
SELECT * FROM stu WHERE english IS NOT NULL;
```

不要写：

```sql
SELECT * FROM stu WHERE english = NULL;
```

## 模糊查询

| 通配符 | 含义     |
| --- | ------ |
| `_` | 单个任意字符 |
| `%` | 任意多个字符 |

示例：

```sql
SELECT * FROM stu WHERE name LIKE '马%';
SELECT * FROM stu WHERE name LIKE '_花%';
SELECT * FROM stu WHERE name LIKE '%德%';
```

## 排序查询

```sql
SELECT * FROM stu ORDER BY age ASC;
SELECT * FROM stu ORDER BY math DESC;
SELECT * FROM stu ORDER BY math DESC, english ASC;
```

多个排序字段时，前一个字段值相同，后一个字段才继续起作用。

## 聚合函数

| 函数        | 作用   |
| --------- | ---- |
| `COUNT()` | 统计数量 |
| `MAX()`   | 最大值  |
| `MIN()`   | 最小值  |
| `SUM()`   | 求和   |
| `AVG()`   | 平均值  |

示例：

```sql
SELECT COUNT(*) FROM stu;
SELECT MAX(math) FROM stu;
SELECT MIN(math) FROM stu;
SELECT SUM(math) FROM stu;
SELECT AVG(math) FROM stu;
```

聚合函数会忽略 NULL。统计总行数一般使用 `COUNT(*)`，如果有主键且引擎和场景合适，也可使用 `COUNT(id)`。

## 分组查询

```sql
SELECT sex, AVG(math) FROM stu GROUP BY sex;
SELECT sex, AVG(math), COUNT(*) FROM stu GROUP BY sex;
```

分组前过滤：

```sql
SELECT sex, AVG(math), COUNT(*)
FROM stu
WHERE math > 70
GROUP BY sex;
```

分组后过滤：

```sql
SELECT sex, AVG(math), COUNT(*)
FROM stu
WHERE math > 70
GROUP BY sex
HAVING COUNT(*) > 2;
```

`WHERE` 和 `HAVING` 区别：

| 项目       | WHERE | HAVING |
| -------- | ----- | ------ |
| 执行时机     | 分组前   | 分组后    |
| 能否使用聚合函数 | 不能直接用 | 可以     |
| 过滤对象     | 原始行   | 分组结果   |

## 分页查询

```sql
SELECT * FROM stu LIMIT 0, 3;
SELECT * FROM stu LIMIT 3, 10;
SELECT * FROM stu LIMIT 100;
```

起始索引从 0 开始。

分页公式：

```text
起始索引 = (页码 - 1) * 每页条数
```

例如第 3 页，每页 20 条：

```sql
SELECT * FROM stu LIMIT 40, 20;
```

大偏移分页性能可能变差，后续可以使用基于主键游标的方式优化：

```sql
SELECT * FROM stu WHERE id > 10000 ORDER BY id LIMIT 20;
```

## 命令速查

| 目的      | SQL                                                      |
| ------- | -------------------------------------------------------- |
| 查看数据库   | `SHOW DATABASES;`                                        |
| 使用数据库   | `USE db_name;`                                           |
| 查看当前数据库 | `SELECT DATABASE();`                                     |
| 查看表     | `SHOW TABLES;`                                           |
| 查看表结构   | `DESC table_name;`                                       |
| 创建数据库   | `CREATE DATABASE db_name DEFAULT CHARACTER SET utf8mb4;` |
| 删除表     | `DROP TABLE IF EXISTS table_name;`                       |
| 插入数据    | `INSERT INTO table_name (...) VALUES (...);`             |
| 修改数据    | `UPDATE table_name SET ... WHERE ...;`                   |
| 删除数据    | `DELETE FROM table_name WHERE ...;`                      |
| 查询数据    | `SELECT ... FROM ... WHERE ...;`                         |

## 易错点

| 问题                 | 处理                             |
| ------------------ | ------------------------------ |
| 字符乱码               | 统一服务端、客户端、库、表为 `utf8mb4`       |
| `UPDATE` 忘记 WHERE  | 会修改全表                          |
| `DELETE` 忘记 WHERE  | 会删除全表数据                        |
| `NULL` 用 `=` 判断    | 必须使用 `IS NULL` / `IS NOT NULL` |
| `GROUP BY` 查询非分组字段 | 结果无确定含义，严格模式下可能报错              |
| `FLOAT/DOUBLE` 存金额 | 使用 `DECIMAL`                   |
| 远程 root 开放 `%`     | 正式环境应限制 IP 或使用专用用户             |

## 关联条目

| 类型 | 条目                                                |
| -- | ------------------------------------------------- |
| 前置 | [[Linux 软件包管理]]、[[RPM]]、[[YUM]]、[[字符集]]           |
| 核心 | [[MySQL]]、[[SQL]]、[[DDL]]、[[DML]]、[[DQL]]         |
| 延伸 | [[MySQL 约束]]、[[多表查询]]、[[MySQL 事务]]、[[MySQL 备份恢复]] |
