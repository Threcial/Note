---
type: Practice
status: Organized
Belongs to: "[[MySQL]]"
Related to:
  - "[[Linux 软件包管理]]"
Has:
  - "[[MySQL 字符集]]"
source: day19.md
created: 2026-06-14
updated: 2026-06-14
---

# MySQL 安装与配置

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

## 配置文件

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

| 参数 | 作用 |
| --- | --- |
| `max_connections` | 最大连接数 |
| `max_allowed_packet` | 单次通信包大小 |
| `skip_name_resolve` | 跳过反向 DNS 解析 |
| `character-set-server` | 服务端默认字符集 |

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

```sql
CREATE USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '复杂密码';
GRANT ALL ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

更安全的写法是限制来源 IP：

```sql
CREATE USER 'app_admin'@'192.168.31.%' IDENTIFIED BY '复杂密码';
GRANT ALL ON app_db.* TO 'app_admin'@'192.168.31.%';
```

## 字符集

创建数据库时明确指定：

```sql
CREATE DATABASE app_db
  DEFAULT CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

| 项目 | 建议 |
| --- | --- |
| 字符集 | `utf8mb4` |
| 通用排序 | `utf8mb4_unicode_ci` |

查看字符集：

```sql
SHOW VARIABLES LIKE '%chara%';
SHOW VARIABLES LIKE '%coll%';
```

## 数据目录迁移

```bash
systemctl stop mysqld
mkdir -p /data/mysql
chown -R mysql:mysql /data
cd /var/lib/mysql
cp -a * /data/mysql
```

修改 `/etc/my.cnf`：

```ini
[mysqld]
datadir=/data/mysql
```

确认权限后重启。

## 易错点

| 问题 | 处理 |
| --- | --- |
| 字符乱码 | 统一服务端、客户端、库、表为 `utf8mb4` |
| root 远程开放 `%` | 限制来源 IP 或使用专用用户 |
| 改 datadir 后启动失败 | 检查权限、SELinux、路径 |
