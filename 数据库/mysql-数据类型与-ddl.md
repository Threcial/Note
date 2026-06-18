---
type: Reference
status: Organized
Belongs to: "[[MySQL]]"
Related to:
  - "[[SQL]]"
Has:
  - "[[MySQL 数据类型]]"
  - "[[DDL]]"
source: day19.md
created: 2026-06-14
updated: 2026-06-14
---

# MySQL 数据类型与 DDL

## SQL 分类

| 分类 | 全称 | 作用 |
| --- | --- | --- |
| DDL | Data Definition Language | 定义库、表、字段 |
| DML | Data Manipulation Language | 插入、修改、删除数据 |
| DQL | Data Query Language | 查询数据 |
| DCL | Data Control Language | 用户、权限、安全控制 |

## 数据类型

### 整数

| 类型 | 字节 |
| --- | --- |
| `TINYINT` | 1 |
| `INT` | 4 |
| `BIGINT` | 8 |

### 小数

| 类型 | 特点 |
| --- | --- |
| `FLOAT` | 近似值，不推荐精确计算 |
| `DOUBLE` | 近似值 |
| `DECIMAL(m,d)` | 精确小数，适合金额 |

### 字符串

| 类型 | 特点 |
| --- | --- |
| `CHAR(n)` | 固定长度，最大 255 |
| `VARCHAR(n)` | 可变长度，常用 |
| `TEXT` | 长文本，最大 65535 |

### 时间

| 类型 | 特点 |
| --- | --- |
| `DATE` | 年月日 |
| `DATETIME` | 年月日时分秒，与时区无关 |
| `TIMESTAMP` | 与时区相关，范围到 2038 年 |

## DDL：数据库操作

```sql
SHOW DATABASES;
CREATE DATABASE app_db;
DROP DATABASE IF EXISTS app_db;
USE app_db;
SELECT DATABASE();
```

## DDL：表操作

```sql
SHOW TABLES;
DESC 表名;

CREATE TABLE student (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(10),
    gender CHAR(1),
    birthday DATE,
    score DECIMAL(5,2),
    email VARCHAR(64),
    status TINYINT
);

DROP TABLE IF EXISTS student;
ALTER TABLE student RENAME TO stu;
ALTER TABLE stu ADD address VARCHAR(50);
ALTER TABLE stu MODIFY address CHAR(50);
ALTER TABLE stu CHANGE address addr VARCHAR(50);
ALTER TABLE stu DROP addr;
```

## 易错点

| 问题 | 处理 |
| --- | --- |
| `FLOAT/DOUBLE` 存金额 | 使用 `DECIMAL` |
| 远程 root 开放 `%` | 正式环境应限制 IP |
