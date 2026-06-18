---
type: Concept
status: Organized
Belongs to: "[[Linux]]"
Related to:
  - "[[权限管理]]"
Has:
  - "[[useradd]]"
  - "[[passwd]]"
  - "[[chmod]]"
  - "[[chown]]"
  - "[[chattr]]"
  - "[[umask]]"
  - "[[SUID]]"
  - "[[SGID]]"
  - "[[粘滞位]]"
  - "[[sudo]]"
source: ""
created: 2026-06-13
updated: 2026-06-13
---

# Linux 用户与权限

Linux 中所有文件和进程都必须有所有者。系统识别用户主要依赖 UID。

## 用户类型

| 类型 | UID 范围 | 说明 |
| --- | --- | --- |
| root | 0 | 超级管理员 |
| 虚拟用户 | 通常不允许登录 | 用于服务进程或文件属主 |
| 普通用户 | CentOS 7 默认从 1000 起 | 可登录 |

## 用户管理基础命令

```bash
id
id 用户名
useradd 用户名
passwd 用户名
userdel -r 用户名
```

## 用户组管理

```bash
groupadd test
groupdel test
gpasswd -a 用户 组   # 把用户加入组
gpasswd -d 用户 组   # 把用户移出组
```

## /etc/passwd

格式：`root:x:0:0:root:/root:/bin/bash`，共 7 列：用户名、密码占位、UID、GID、说明、家目录、Shell。

## /etc/shadow

存放加密密码，共 9 列：用户名、加密密码（SHA512）、最近修改时间、最小间隔、有效期、警告天数、宽限天数、失效时间、保留。

## useradd 进阶

```bash
useradd hhh -u 888 -s /sbin/nologin -M -c xuniuser
```

| 选项 | 含义 |
| --- | --- |
| `-u` | 指定 UID |
| `-c` | 添加说明 |
| `-d` | 指定家目录 |
| `-s` | 指定 Shell |
| `-G` | 加入附加组 |
| `-g` | 指定初始组 |
| `-M` | 不创建家目录 |

默认配置参考 `/etc/default/useradd` 和 `/etc/login.defs`。

## passwd 进阶

```bash
passwd -l test2       # 锁定用户
passwd -u test2       # 解锁用户
echo "111111" | passwd --stdin zl102
```

## su

```bash
su - cjk
```

`-` 表示切换用户身份的同时切换登录环境变量。

## 文件权限

```text
-rwxrwxrwx 1 root root 25 Dec 17 10:06 hello.py
```

### 文件类型位

| 字符 | 含义 |
| --- | --- |
| `d` | 目录 |
| `-` | 普通文件 |
| `l` | 软链接 |
| `b` | 块设备 |
| `c` | 字符设备 |
| `s` | socket |

### 权限位

```text
rwx rwx rwx
属主 属组 其他人
```

| 字母 | 含义 | 数字 |
| --- | --- | --- |
| `r` | 读 | 4 |
| `w` | 写 | 2 |
| `x` | 执行 | 1 |

### 常见权限

| 权限 | 数字 | 场景 |
| --- | --- | --- |
| `drwxr-xr-x` | 755 | 目录常见默认权限 |
| `-rw-r--r--` | 644 | 文件常见默认权限 |

## chmod

```bash
chmod u+x test.txt
chmod 640 test.txt
chmod -R 755 dir/
chmod --reference ref.txt target.txt
```

## 文件与目录上的 rwx

### 文件

| 权限 | 含义 |
| --- | --- |
| `r` | 查看文件内容 |
| `w` | 修改文件内容 |
| `x` | 文件可被执行 |

删除文件看上级目录的写权限，不是看文件本身的 `w`。

### 目录

| 权限 | 含义 |
| --- | --- |
| `r` | 列出目录中文件名 |
| `w` | 在目录中创建、删除、移动 |
| `x` | 进入目录、访问目录下具体路径 |

目录通常至少需要 `r+x` 才能正常查看内容。

## chown

```bash
chown 用户:用户组 文件
chown -R 用户:用户组 目录
```

## chattr 与 lsattr

```bash
chattr +i /etc/passwd
lsattr /etc/passwd
chattr -i /etc/passwd
```

`+i` 让文件不可被修改、删除、重命名，即使 root 也需要先取消。

## umask

```bash
umask
umask -S
umask 033
```

文件默认最高权限 `666`，目录默认最高权限 `777`。

## 特殊权限

| 特殊权限 | 数字 | 典型场景 |
| --- | --- | --- |
| SUID | 4 | `/usr/bin/passwd` |
| SGID | 2 | 组继承 |
| 粘滞位 | 1 | `/tmp` |

粘滞位让所有用户可创建文件，但不能随意删除其他用户的文件。

## sudo

```bash
visudo
visudo -c
```

### sudoers 格式

```text
root    ALL=(ALL)       ALL
cjk     ALL=(ALL)       /usr/sbin/useradd, /usr/bin/passwd, !/usr/bin/passwd root
```

| 字段 | 含义 |
| --- | --- |
| 第 1 段 | 用户名或用户组（组前加 `%`） |
| 第 2 段 | 主机，通常 `ALL` |
| 第 3 段 | 可切换身份 |
| 第 4 段 | 允许执行的命令 |

`NOPASSWD: ALL` 表示执行 sudo 不需要密码，权限很大，应谨慎使用。

```bash
sudo -l
```

## 易错点

| 场景 | 说明 |
| --- | --- |
| 删除文件看文件本身 `w` | 删除了看上级目录 `w+x` |
| 目录只有 `r` 就能正常访问 | 没有 `x` 不能进入和访问详细信息 |
| `sudo ALL` 给普通用户 | 风险接近 root |
| `chattr +i` 后无法改文件 | 需要先 `chattr -i` |
