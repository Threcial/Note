---
title: Linux 用户、权限、sudo 与磁盘文件系统基础
type: knowledge
status: evergreen
source: day03.md
created: 2026-06-13
updated: 2026-06-13
tags:
  - Linux
  - CentOS7
  - 用户管理
  - 用户组
  - 权限管理
  - sudo
  - 磁盘
  - 文件系统
  - mount
  - swap
aliases:
  - DAY03
  - Linux 用户权限
  - Linux sudo 授权
  - Linux 磁盘基础
Belongs to:
  - "[[Linux 运维基础]]"
  - "[[权限管理]]"
  - "[[磁盘管理]]"
Related to:
  - "[[day01 CentOS 7 基础环境与常用命令]]"
  - "[[day02 管道、xargs、启动流程与系统核心文件]]"
  - "[[Linux 文件属性]]"
  - "[[Linux 文件系统]]"
  - "[[fstab]]"
Has:
  - "[[useradd]]"
  - "[[passwd]]"
  - "[[gpasswd]]"
  - "[[chmod]]"
  - "[[chown]]"
  - "[[chattr]]"
  - "[[umask]]"
  - "[[SUID]]"
  - "[[SGID]]"
  - "[[粘滞位]]"
  - "[[sudo]]"
  - "[[inode]]"
  - "[[软链接]]"
  - "[[硬链接]]"
  - "[[MBR]]"
  - "[[mount]]"
  - "[[swap]]"
_organized: true
---
# Linux 用户、权限、sudo 与磁盘文件系统基础

## 概览

- Linux 中所有文件和进程都必须有所有者，用户和用户组是权限体系的基础。
- 系统识别用户主要依赖 UID，用户名只是便于管理的显示名称。
- 文件权限的 `rwx` 对文件和目录含义不同，删除文件看的是上级目录的写权限。
- `sudo` 应按命令精确授权，授权越粗，风险越大。
- 磁盘要使用，一般经历分区、格式化、挂载三个阶段。
- inode 保存文件元数据和数据块位置，硬链接共享 inode，软链接保存路径指向。

## 关系表

| 条目     | 上游依赖            | 下游用途                |
| ------ | --------------- | ------------------- |
| 用户/用户组 | UID、GID、进程所有者   | 文件归属、登录控制、服务运行身份    |
| 文件权限   | 文件类型、属主、属组      | 访问控制、服务权限排错         |
| sudo   | 普通用户、命令路径       | 最小权限授权、运维账号管理       |
| inode  | 文件系统、block      | 硬链接、文件定位、inode 耗尽排查 |
| mount  | 设备文件、文件系统       | 挂载硬盘、光盘、U 盘         |
| fstab  | UUID、挂载点、文件系统类型 | 开机自动挂载              |

## 用户与用户组

### 用户类型

| 类型   | UID 范围/特征           | 说明                        |
| ---- | ------------------- | ------------------------- |
| root | UID 为 0             | 超级管理员，系统中只有 UID=0 代表最高权限  |
| 虚拟用户 | 通常不允许登录             | 用于满足服务进程或文件属主需求，降低管理风险    |
| 普通用户 | CentOS 7 默认从 1000 起 | 可登录，权限通常限制在家目录和允许访问的系统范围内 |

用户信息文件：

```bash
/etc/passwd
```

### 用户组

用户组是用户集合。默认创建普通用户时，系统通常会自动创建一个同名主组。

用户和用户组不是一对一关系：

- 一个用户可以属于多个组。
- 多个用户可以属于同一个组。
- 多个用户和多个用户组可以形成多对多关系。

## 用户管理基础命令

```bash
id
id 用户名
whoami
useradd 用户名
passwd 用户名
passwd
userdel -r 用户名
```

| 命令           | 作用                 |
| ------------ | ------------------ |
| `id`         | 查看当前用户 UID/GID/组信息 |
| `id user`    | 查看指定用户信息           |
| `whoami`     | 查看当前登录用户           |
| `useradd`    | 新建用户               |
| `passwd`     | 修改密码               |
| `userdel -r` | 删除用户并删除家目录         |

## 用户组管理命令

```bash
groupadd test
tail -n 1 /etc/group
groupdel test
```

用户组可以单独存在。

## /etc/passwd

格式示例：

```typescript
root:x:0:0:root:/root:/bin/bash
apache:x:48:48:Apache:/usr/share/httpd:/sbin/nologin
xwx:x:1003:1003::/home/xwx:/bin/bash
```

共 7 列：

| 列 | 含义                             |
| - | ------------------------------ |
| 1 | 用户名                            |
| 2 | 密码占位，`x` 表示密码在 `/etc/shadow` 中 |
| 3 | UID                            |
| 4 | GID                            |
| 5 | 用户说明                           |
| 6 | 家目录                            |
| 7 | Shell 解释器                      |

`/sbin/nologin` 表示禁止交互登录，常用于服务用户。

## /etc/shadow

格式示例：

```yaml
root:$6$...:18372:0:99999:7:::
wb:!!:18562:0:99999:7:::
```

共 9 列：

| 列 | 含义                         |
| - | -------------------------- |
| 1 | 用户名                        |
| 2 | 加密后的密码，CentOS 常见 SHA512    |
| 3 | 最近修改密码时间，自 1970-01-01 起的天数 |
| 4 | 两次密码修改最小间隔                 |
| 5 | 密码有效期                      |
| 6 | 密码过期前警告天数                  |
| 7 | 密码过期后的宽限天数                 |
| 8 | 账号失效时间                     |
| 9 | 保留字段                       |

时间换算：

```bash
date -d "1970-01-01 18562 days"
```

## /etc/group

格式示例：

```yaml
root:x:0:
wheel:x:10:centos
```

共 4 列：

| 列 | 含义   |
| - | ---- |
| 1 | 用户组名 |
| 2 | 组密码位 |
| 3 | GID  |
| 4 | 组成员  |

## useradd 进阶

```bash
useradd hhh -u 888 -s /sbin/nologin -M -c xuniuser
useradd -s /sbin/nologin -M mysql
```

| 选项   | 含义       |
| ---- | -------- |
| `-u` | 指定 UID   |
| `-c` | 添加说明     |
| `-d` | 指定家目录    |
| `-s` | 指定 Shell |
| `-G` | 加入附加组    |
| `-g` | 指定初始组    |
| `-M` | 不创建家目录   |

默认配置参考：

```bash
/etc/default/useradd
/etc/login.defs
```

## passwd 进阶

```bash
passwd -l test2
passwd -u test2
echo "111111" | passwd --stdin zl102
```

| 选项        | 含义                |
| --------- | ----------------- |
| `-l`      | 锁定用户密码            |
| `-u`      | 解锁用户密码            |
| `--stdin` | 从标准输入读取密码，适合脚本化设置 |

## su 切换用户

```bash
su - cjk
```

`-` 表示切换用户身份的同时切换登录环境变量。

| 场景      | 是否需要密码      |
| ------- | ----------- |
| 高权限切低权限 | 通常不需要       |
| 低权限切高权限 | 需要目标高权限用户密码 |

## gpasswd 管理附加组

```bash
gpasswd -a cjk av
gpasswd -d cjk av
id cjk
```

| 选项        | 含义     |
| --------- | ------ |
| `-a 用户 组` | 把用户加入组 |
| `-d 用户 组` | 把用户移出组 |

## useradd 默认配置

### /etc/default/useradd

```ini
GROUP=100
HOME=/home
INACTIVE=-1
EXPIRE=
SHELL=/bin/bash
SKEL=/etc/skel
CREATE_MAIL_SPOOL=yes
```

| 配置                  | 含义                    |
| ------------------- | --------------------- |
| `GROUP`             | 默认初始组策略               |
| `HOME`              | 家目录父目录                |
| `INACTIVE`          | 密码过期后的宽限天数，`-1` 表示不过期 |
| `EXPIRE`            | 账号失效时间                |
| `SHELL`             | 默认 Shell              |
| `SKEL`              | 新用户家目录模板              |
| `CREATE_MAIL_SPOOL` | 是否创建邮箱文件              |

### /etc/skel

`/etc/skel` 是新用户家目录模板。创建用户时，该目录下的文件会复制到新用户家目录。

实验逻辑：

```bash
cd /etc/skel/
touch hello
useradd test222
su - test222
ls -a
```

新用户家目录中会出现 `hello`。

### /etc/login.defs

```ini
MAIL_DIR        /var/spool/mail
PASS_MAX_DAYS   99999
PASS_MIN_DAYS   0
PASS_MIN_LEN    5
PASS_WARN_AGE   7
UID_MIN         1000
UID_MAX         60000
CREATE_HOME     yes
UMASK           077
USERGROUPS_ENAB yes
ENCRYPT_METHOD SHA512
```

| 配置                    | 含义                 |
| --------------------- | ------------------ |
| `MAIL_DIR`            | 邮箱位置               |
| `PASS_MAX_DAYS`       | 密码有效期              |
| `PASS_MIN_DAYS`       | 密码最小修改间隔           |
| `PASS_MIN_LEN`        | 密码最小长度，实际多由 PAM 接管 |
| `PASS_WARN_AGE`       | 到期前警告天数            |
| `UID_MIN` / `UID_MAX` | 普通用户 UID 范围        |
| `CREATE_HOME`         | 是否创建家目录            |
| `UMASK`               | 默认权限掩码             |
| `USERGROUPS_ENAB`     | 删除用户时是否删除同名用户组     |
| `ENCRYPT_METHOD`      | 密码加密方式             |

## 时间同步补充

系统时间异常时，先确认时区。能联网时可同步网络时间：

```bash
ntpdate ntp3.aliyun.com
hwclock -w
```

不能联网时手动设置：

```bash
date -s "月份/日期/年份 小时:分钟:秒"
hwclock -w
```

`hwclock -w` 会把系统时间写入硬件时钟，避免重启后时间再次错误。

## 命名规范

文件、目录、变量、用户等名称应有明确含义，避免使用 `a`、`b`、`1`、`2` 这类无法表达用途的名称。

## Linux 文件权限

示例：

```text
-rwxrwxrwx 1 root root 25 Dec 17 10:06 hello.py
```

### 文件类型位

| 字符  | 含义     |
| --- | ------ |
| `d` | 目录     |
| `-` | 普通文件   |
| `l` | 软链接    |
| `b` | 块设备    |
| `c` | 字符设备   |
| `s` | socket |

### 权限位

9 个权限位分为三组：

```text
rwx rwx rwx
属主 属组 其他人
```

| 字母  | 含义         | 数字 |
| --- | ---------- | -- |
| `r` | read，读     | 4  |
| `w` | write，写    | 2  |
| `x` | execute，执行 | 1  |
| `-` | 无权限        | 0  |

对象简写：

| 简写  | 含义        |
| --- | --------- |
| `u` | user，属主   |
| `g` | group，属组  |
| `o` | other，其他人 |
| `a` | all，全部    |

常见权限：

| 权限           | 数字  | 场景       |
| ------------ | --- | -------- |
| `drwxr-xr-x` | 755 | 目录常见默认权限 |
| `drwxrwxrwx` | 777 | 目录最大普通权限 |
| `-rw-r--r--` | 644 | 文件常见默认权限 |
| `-rwxrwxrwx` | 777 | 文件最大普通权限 |

## chmod

```bash
chmod [选项] [符号权限或数字权限] 文件/目录
```

常用选项：

| 选项                 | 含义            |
| ------------------ | ------------- |
| `-R`               | 递归修改          |
| `--reference=参考文件` | 按参考文件权限设置目标文件 |

示例：

```bash
chmod u+x test.txt
chmod a-r test.txt
chmod u=rwx,g=rx,o=r time.txt
chmod 640 test.txt
chmod -R 755 weibo/
chmod --reference 1.txt 111111
```

## 文件与目录上的 rwx 含义

### 文件

| 权限  | 含义     | 示例命令                              |
| --- | ------ | --------------------------------- |
| `r` | 查看文件内容 | `cat`、`less`、`more`               |
| `w` | 修改文件内容 | `vim`、`echo >`、`echo >>`          |
| `x` | 文件可被执行 | `/bin/sh script.sh`、`./script.sh` |

删除文件不看文件本身的 `w`，而看上级目录是否有写权限。

### 目录

| 权限  | 含义             | 示例命令                           |
| --- | -------------- | ------------------------------ |
| `r` | 列出目录中文件名       | `ls`                           |
| `w` | 在目录中创建、删除、移动   | `touch`、`mkdir`、`rm`、`mv`、`cp` |
| `x` | 进入目录、访问目录下具体路径 | `cd`                           |

目录通常至少需要 `r+x` 才能正常查看内容。只有 `r` 没有 `x` 时，能看到文件名但无法读取详细属性。

## 权限实验结论

实验环境：用户 `cjk`，目录 `123`，文件 `123/abc`。

### 目录权限影响

| 目录权限       | 现象                                 |
| ---------- | ---------------------------------- |
| 无权限        | `ls` 和 `cd` 都失败                    |
| 只有 `r`     | 可看到文件名，但详细属性显示 `?????????`，不能 `cd` |
| `rw` 无 `x` | 仍不能进入目录                            |
| 加上 `x`     | 可以进入目录并查看文件属性                      |

### 文件权限影响

| 文件权限      | 现象            |
| --------- | ------------- |
| 无属主权限     | `cat` 和写入失败   |
| 加 `r`     | 可读取，但不能写入     |
| 加 `w`     | 可写入内容，但不能删除文件 |
| 上级目录加 `w` | 可删除文件         |

## chown 与 chgrp

### chown

```bash
chown 用户:用户组 文件
chown 用户 文件
chown :用户组 文件
chown -R 用户:用户组 目录
```

示例：

```bash
chown bdyjy test.txt
chown bdyjy:av weibo/
```

### chgrp

```bash
chgrp 用户组 文件/目录
chgrp -R 用户组 目录
```

示例：

```bash
chgrp av test.txt
```

`chgrp` 只改属组，用得比 `chown` 少。

## chattr 与 lsattr

```bash
chattr +i /etc/passwd
lsattr /etc/passwd
chattr -i /etc/passwd
```

`+i` 会让文件不可被修改、删除、重命名，即使 root 也需要先取消该属性。常用于保护关键文件，例如 `/etc/passwd`、`/etc/shadow`。

## umask

```bash
umask
umask -S
umask 033
```

文件默认最高权限是 `666`，目录默认最高权限是 `777`。文件默认不会自动带执行权限。

> **注意**\
> umask 不能简单用数字减法理解，需要按权限位字符计算。原始记录中的 `777 - 033 = 744`、`666 - 033 = 633` 这种算式不是严谨计算方式。

## 特殊权限

Linux 基础权限是 9 位，额外还有 SUID、SGID、粘滞位。

| 特殊权限 | 数字 | 位置        | 有执行位显示 | 无执行位显示 | 典型场景              |
| ---- | -- | --------- | ------ | ------ | ----------------- |
| SUID | 4  | 属主 `x` 位  | `s`    | `S`    | `/usr/bin/passwd` |
| SGID | 2  | 属组 `x` 位  | `s`    | `S`    | 组继承、部分命令权限        |
| 粘滞位  | 1  | 其他人 `x` 位 | `t`    | `T`    | `/tmp`            |

查看 `/tmp`：

```bash
ls -ld /tmp/
```

典型权限：

```text
drwxrwxrwt
```

粘滞位让所有用户可在 `/tmp` 中创建文件，但不能随意删除其他用户的文件。

## sudo 授权

### 基本概念

`sudo` 用于给普通用户授予指定管理员权限。权限越精确，风险越低；权限越宽泛，普通用户越接近 root。

编辑配置：

```bash
visudo
visudo -c
```

`visudo` 等价于安全编辑 `/etc/sudoers`，会进行语法检查。

### sudoers 格式

```text
root    ALL=(ALL)       ALL
byf     ALL=(ALL)       /usr/bin/rm
```

| 字段    | 含义                     |
| ----- | ---------------------- |
| 第 1 段 | 用户名或用户组，用户组前加 `%`      |
| 第 2 段 | 主机，通常写 `ALL`           |
| 第 3 段 | 可切换成什么身份，`ALL` 包含 root |
| 第 4 段 | 允许执行的命令，`ALL` 表示所有命令   |

### 授权重启

```bash
which shutdown
visudo
```

```sudoers
cjk     ALL=(ALL)   /usr/sbin/shutdown -r now
```

验证：

```bash
su - cjk
sudo -l
sudo /usr/sbin/shutdown -r now
sudo /usr/sbin/shutdown -h now
```

只授权了 `shutdown -r now`，执行 `shutdown -h now` 会被拒绝。

### 授权创建用户但禁止改 root 密码

```bash
which useradd
which passwd
```

```sudoers
cjk     ALL=(ALL)   /usr/sbin/useradd, /usr/bin/passwd, !/usr/bin/passwd root
```

验证：

```bash
sudo /usr/sbin/useradd byf
sudo /usr/bin/passwd root
sudo /usr/bin/passwd byf
```

### 授予管理员权限

```sudoers
cjk     ALL=(ALL)  NOPASSWD: ALL
```

`NOPASSWD` 表示执行 sudo 时不需要输入当前用户密码。该配置权限很大，应谨慎使用。

## 硬盘结构基础

### 存储设备类型

常见外部存储设备包括硬盘、光盘、U 盘、SAN、NAS 等。

硬盘按介质可分为：

| 类型       | 特点                    |
| -------- | --------------------- |
| 机械硬盘 HDD | 磁性碟片存储，怕震动，随机读写慢      |
| 固态硬盘 SSD | 闪存颗粒存储，随机读写快，寿命与写入量相关 |

SSD 常见寿命指标：

```text
100TBW = 总写入量约 100TB
1PBW = 1000TBW
```

选购 SSD 时要考虑业务每天实际写入量。

性能结论：

- 顺序读写：SSD 和 HDD 差距可能不如随机读写明显。
- 随机读写：SSD 通常比 HDD 快很多。

参考链接保留：

```text
机械硬盘：https://zhuanlan.zhihu.com/p/27926239
机械硬盘：https://zhuanlan.zhihu.com/p/89505052
固态硬盘：https://zhuanlan.zhihu.com/p/43362595
固态硬盘：https://www.cnblogs.com/ricklz/p/16415763.html
inode：https://www.cnblogs.com/wanng/p/linux-inodes.html
```

## inode、block 与文件名

Linux 文件系统可理解为三部分：

| 组成    | 作用                                     |
| ----- | -------------------------------------- |
| 文件名   | 便于用户识别，本质上关联 inode 号                   |
| inode | 保存文件元数据，例如大小、属主、属组、权限、时间戳、链接数、block 位置 |
| block | 真正保存文件数据                               |

查看 inode：

```bash
ls -i 文件
```

查看 inode 使用情况：

```bash
df -i
```

inode 在格式化文件系统时分配，数量固定或按规则生成。小文件极多时，可能出现磁盘空间没满但 inode 耗尽的情况。

## 软链接

```bash
ln -s 原地址 目标地址
ln -s sanguo sg
```

软链接类似 Windows 快捷方式，可链接文件，也可链接目录。

| 操作           | 结果                                |
| ------------ | --------------------------------- |
| 删除原文件        | 软链接变成死链接                          |
| `rm -rf sg/` | 如果 `sg` 指向目录，可能删除目标目录中的内容，软链接本身还在 |
| `rm -rf sg`  | 删除软链接本身                           |

## 硬链接

```bash
ln 原地址 目标地址
ln c d
ls -i
```

硬链接特点：

- 只能链接文件，不能直接链接目录。
- 原文件和硬链接文件共享同一个 inode。
- 修改任意一个，另一个看到的内容同步变化。
- 删除其中一个，不影响另一个继续访问数据。
- 不额外占用一份完整数据空间。

## 磁盘逻辑结构

机械硬盘逻辑结构：

| 结构 | 含义                    |
| -- | --------------------- |
| 磁道 | 盘片上的同心圆逻辑区域           |
| 扇区 | 磁道上的弧段，传统大小常见 512Byte |
| 柱面 | 多个盘面上编号相同的磁道组成的圆柱     |

传统硬盘容量计算模型：

```text
磁头数 × 柱面数 × 每柱面扇区数 × 每扇区大小
```

## 设备命名

| 场景                      | 常见设备名                    |
| ----------------------- | ------------------------ |
| IDE / SATA / SCSI / SAS | `/dev/sda`、`/dev/sdb`    |
| 云服务器虚拟磁盘                | `/dev/vda`、`/dev/xvda` 等 |

## MBR 分区

MBR 位于硬盘的第一个扇区，大小 512 字节。

| 区域      | 大小     | 作用         |
| ------- | ------ | ---------- |
| 主引导程序   | 446 字节 | 引导加载       |
| DPT 分区表 | 64 字节  | 记录分区信息     |
| 有效标志    | 2 字节   | 标识 MBR 有效性 |

MBR 分区限制：

- 主分区最多 4 个。
- 逻辑分区必须建立在扩展分区之上。
- 扩展分区本身不能直接存数据，只是逻辑分区容器。

## fstab

文件：

```bash
/etc/fstab
```

常见挂载选项：

| 选项                  | 含义                                        |
| ------------------- | ----------------------------------------- |
| `async`             | 异步写入，常见默认模式                               |
| `sync`              | 同步写入，性能开销大                                |
| `atime` / `noatime` | 是否更新访问时间                                  |
| `auto` / `noauto`   | `mount -a` 时是否自动挂载                        |
| `dev` / `nodev`     | 是否允许设备文件                                  |
| `exec` / `noexec`   | 是否允许执行文件                                  |
| `suid` / `nosuid`   | 是否允许 SUID/SGID 生效                         |
| `user` / `nouser`   | 普通用户是否可挂载/卸载                              |
| `remount`           | 重新挂载                                      |
| `ro`                | 只读                                        |
| `rw`                | 可读写                                       |
| `loop`              | 把文件作为块设备挂载                                |
| `defaults`          | 默认选项，通常包含 `rw,suid,dev,exec,nouser,async` |

最后两列：

| 字段      | 值   | 含义      |
| ------- | --- | ------- |
| dump 备份 | `0` | 不备份     |
| dump 备份 | `1` | 备份      |
| dump 备份 | `2` | 不定期备份   |
| fsck 检查 | `0` | 不检查     |
| fsck 检查 | `1` | 启动时优先检查 |
| fsck 检查 | `2` | 启动后检查   |

## 文件系统类型

| 类型       | 说明                          |
| -------- | --------------------------- |
| Windows  | NTFS、FAT32                  |
| Linux    | ext3、ext4、xfs               |
| 网络共享文件系统 | NFS、SMB                     |
| 集群文件系统   | GFS、OCFS                    |
| 分布式文件系统  | Ceph、HDFS                   |
| VFS      | Virtual File System，虚拟文件系统层 |

CentOS 7 默认常用 XFS 日志型文件系统。

## 为什么要格式化

磁盘要真正存放文件，需要创建文件系统。完整流程通常是：

```text
分区 -> 格式化/创建文件系统 -> 挂载
```

格式化不是简单清空，而是建立文件系统结构，例如 inode 区、数据区和元数据规则。

## 虚拟机添加磁盘并分区

### 设备识别

```bash
ls /dev/sd*
fdisk -l /dev/sdb
```

### fdisk 分区

```bash
fdisk /dev/sdb
```

常用交互命令：

| 命令  | 含义     |
| --- | ------ |
| `m` | 帮助     |
| `n` | 新建分区   |
| `p` | 打印分区表  |
| `d` | 删除分区   |
| `l` | 列出分区类型 |
| `t` | 修改分区类型 |
| `q` | 不保存退出  |
| `w` | 保存并退出  |

创建 5G 主分区示例：

```text
Command (m for help): n
Select (default p): p
Partition number (1-4, default 1): 1
First sector: 回车
Last sector: +5G
Command (m for help): w
```

### 格式化与挂载

```bash
mkfs.xfs /dev/sdb1
mkdir /test
mount /dev/sdb1 /test/
df -h
umount /test/
```

永久挂载：

```bash
blkid
vim /etc/fstab
```

```fstab
UUID="271074c0-28bd-4ccb-b8ff-6076b276b7f2" /test xfs defaults 0 0
```

## 分区规划示例

1T 磁盘参考：

| 挂载点     | 大小   | 文件系统 | 用途            |
| ------- | ---- | ---- | ------------- |
| `/boot` | 2G   | xfs  | 启动文件          |
| `swap`  | 16G  | swap | 交换空间          |
| `/`     | 200G | xfs  | 系统根目录         |
| `/data` | 800G | xfs  | 数据库文件、代码、业务数据 |

通用服务器可以只划一个较大的根分区；数据库或代码服务器更适合单独规划 `/data`。

## swap 扩展

### 使用分区扩展 swap

```bash
mkswap -f /dev/sdb6
free -h
swapon /dev/sdb6
free -h
swapoff /dev/sdb6
```

永久生效需要写入 `/etc/fstab`。

### 使用文件扩展 swap

```bash
dd if=/dev/zero of=swap_file bs=1M count=500
chmod 600 swap_file
mkswap swap_file
swapon swap_file
free -h
swapoff swap_file
```

原始实验中 `swapon` 提示 `insecure permissions 0644, 0600 suggested`，处理方式是给 swap 文件设置 `600` 权限。

## df 与 du

```bash
df -h
du /root/ -sh
```

| 命令          | 作用              |
| ----------- | --------------- |
| `df -h`     | 查看文件系统挂载和空间使用情况 |
| `du -sh 路径` | 统计目录或文件占用大小     |

## mount 挂载

挂载是把设备名和一个已存在目录建立关联。Linux 中硬盘、光盘、U 盘等设备都需要挂载后才能通过目录访问。

### mount 命令

```bash
mount
mount [选项] 设备 挂载点
mount -a
```

| 选项   | 含义                  |
| ---- | ------------------- |
| `-t` | 指定文件系统类型            |
| `-o` | 指定挂载选项              |
| `-a` | 按 `/etc/fstab` 自动挂载 |

### 挂载光盘镜像

```bash
ls /dev/sr*
mount -t iso9660 /dev/sr0 /mnt/cdrom/
ls /mnt/cdrom/
umount /mnt/cdrom
```

如果卸载时报“目标忙”，通常是当前目录还在挂载点内，先切换出去：

```bash
cd ..
umount /mnt/cdrom
```

### 挂载 U 盘

```bash
fdisk -l
mount /dev/sdb1 /mnt/usb/
```

FAT32 中文乱码时：

```bash
umount /mnt/usb
mount -t vfat -o iocharset=utf8 /dev/sdb1 /mnt/usb/
```

NTFS 格式需要安装：

```bash
yum install epel-release -y
yum install ntfs-3g -y
mount -t ntfs -o iocharset=utf8 /dev/sdb1 /mnt/usb/
```

## 命令速查

| 目标           | 命令                                  |
| ------------ | ----------------------------------- |
| 查看用户         | `id user`                           |
| 创建虚拟用户       | `useradd -s /sbin/nologin -M mysql` |
| 加入附加组        | `gpasswd -a 用户 组`                   |
| 修改权限         | `chmod 640 file`                    |
| 修改属主属组       | `chown user:group file`             |
| 查看不可变属性      | `lsattr file`                       |
| 设置不可变        | `chattr +i file`                    |
| 查看 sudo 权限   | `sudo -l`                           |
| 查看 inode     | `ls -i file`                        |
| 查看 inode 使用率 | `df -i`                             |
| 创建软链接        | `ln -s src dst`                     |
| 创建硬链接        | `ln src dst`                        |
| 查看磁盘         | `fdisk -l`                          |
| 格式化 XFS      | `mkfs.xfs /dev/sdb1`                |
| 临时挂载         | `mount /dev/sdb1 /test`             |
| 卸载           | `umount /test`                      |
| 查看空间         | `df -h`                             |

## 易错点

| 场景                  | 说明                   |
| ------------------- | -------------------- |
| 删除文件看文件本身 `w`       | 错。删除看上级目录 `w+x`      |
| 目录只有 `r` 就能正常访问     | 错。没有 `x` 不能进入和访问详细信息 |
| `sudo ALL` 给普通用户    | 风险接近 root，应尽量按命令授权   |
| `chattr +i` 后无法改文件  | 需要先 `chattr -i`      |
| 软链接目录后加 `/` 删除      | 可能删除目标目录内容，不只是链接     |
| 修改 `/etc/fstab` 不测试 | 可能导致开机挂载失败           |
| swap 文件权限 0644      | 应改成 0600             |
| NTFS 直接挂载失败         | CentOS 7 需 `ntfs-3g` |

## 关联条目

- [[Linux 用户与用户组]]
- [[Linux 文件权限]]
- [[sudo 授权]]
- [[inode]]
- [[软链接]]
- [[硬链接]]
- [[MBR]]
- [[fstab]]
- [[mount]]
- [[swap]]
