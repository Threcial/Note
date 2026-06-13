---
title: CentOS 7 基础环境与常用命令
type: knowledge
status: evergreen
created: 2026-06-13
updated: 2026-06-13
tags:
  - Linux
  - CentOS7
  - 运维基础
  - 文件系统
  - Shell
aliases:
  - DAY01
  - Linux 基础命令
  - CentOS7 入门
Belongs to:
  - "[[Linux 运维基础]]"
  - "[[CentOS 7]]"
Related to:
  - "[[VMware]]"
  - "[[Linux 文件系统]]"
  - "[[Shell 重定向]]"
  - "[[Vim]]"
  - "[[find 命令]]"
Has:
  - "[[Linux 目录结构]]"
  - "[[Linux 命令格式]]"
  - "[[ls]]"
  - "[[cd]]"
  - "[[cp]]"
  - "[[mv]]"
  - "[[rm]]"
  - "[[cat]]"
  - "[[less]]"
  - "[[date]]"
  - "[[alias]]"
  - "[[stat]]"
  - "[[locate]]"
_organized: true
---
# CentOS 7 基础环境与常用命令

## 概览

- Linux 的入口只有一个根目录 `/`，没有 Windows 里的 C/D/E 盘概念。
- 命令严格区分大小写，基本格式是 `命令 [选项] [参数]`。
- `root` 的提示符通常是 `#`，普通用户通常是 `$`。
- 目录、文件、命令、权限、重定向、Vim、查找命令是后续 Linux 运维的基础块。
- 危险命令优先理解再执行，尤其是 `rm -rf`、覆盖重定向 `>`、`mv`/`cp` 覆盖同名文件。

## 环境与入口

### 服务器形态

| 类型               | 说明                          |
| ---------------- | --------------------------- |
| 1U / 2U / 4U 服务器 | 机架式服务器，U 表示机柜高度单位           |
| 机柜               | 用于集中安装服务器、交换机、电源等设备         |
| 机房               | 服务器、网络、电源、空调、消防、布线共同组成的运行环境 |
| 机房走线             | 影响后期排障、扩容和可维护性              |

参考链接：

```text
1U服务器：https://item.jd.com/10120657824069.html
2U服务器：https://item.jd.com/100293205158.html?spmTag=YTAyMTkuYjAwMjM1Ni5jMDAwMDQ2ODkuMCUyM2hpc2tleXdvcmQlMkNhMDI0MC5iMDAyNDkzLmMwMDAwNDAyNy41JTIzc2t1X2NhcmQ
4U服务器：https://item.jd.com/10030050991564.html
机柜：https://item.jd.com/10073064809683.html
```

### 云厂商

```text
阿里云：https://www.aliyun.com/
腾讯云：https://cloud.tencent.com/
```

### 虚拟机实验环境

在 Windows 上用 VMware 安装 CentOS 7 虚拟机。

```text
VMware Workstation Player:
https://www.vmware.com/products/workstation-player/workstation-player-evaluation.html.html

CentOS 7 ISO:
https://mirrors.aliyun.com/centos/7/isos/x86_64/CentOS-7-x86_64-DVD-2009.iso
```

Xshell / Xftp 属于远程连接和文件传输工具。安装后删除 `liveupdate.exe`，原因是部分杀毒软件可能误报。

## 登录提示符

登录 CentOS 后常见提示符：

```bash
[root@localhost ~]#
```

| 部分          | 含义          |
| ----------- | ----------- |
| `root`      | 当前登录用户名     |
| `@`         | 用户名和主机名的分隔符 |
| `localhost` | 主机名         |
| `~`         | 当前用户家目录     |
| `#`         | root 用户提示符  |
| `$`         | 普通用户提示符     |

家目录是系统为用户分配的个人工作目录。root 的家目录是 `/root`，普通用户通常位于 `/home/用户名`。

## Linux 目录结构

| 目录       | 作用                                                                       |
| -------- | ------------------------------------------------------------------------ |
| `/`      | 根目录，所有文件的入口                                                              |
| `/bin`   | 普通用户常用命令                                                                 |
| `/sbin`  | 管理员命令                                                                    |
| `/boot`  | 系统启动相关文件                                                                 |
| `/dev`   | 设备文件，硬件会被映射成文件                                                           |
| `/etc`   | 配置文件，运维中非常高频                                                             |
| `/home`  | 普通用户家目录                                                                  |
| `/root`  | root 用户家目录                                                               |
| `/lib`   | 32 位系统库目录                                                                |
| `/lib64` | 64 位系统库目录                                                                |
| `/media` | 系统自动挂载设备的常见位置                                                            |
| `/mnt`   | 用户手动挂载设备的常见位置                                                            |
| `/opt`   | 第三方软件安装目录                                                                |
| `/proc`  | 内核和进程运行状态的虚拟文件系统                                                         |
| `/run`   | 系统运行时数据                                                                  |
| `/srv`   | 服务数据目录                                                                   |
| `/sys`   | 内核设备模型相关虚拟文件系统                                                           |
| `/tmp`   | 临时目录，可能被系统定期清理，不适合保存重要文件                                                 |
| `/usr`   | 系统程序、库、文档等数据。CentOS 7 中 `/bin`、`/sbin`、`/lib`、`/lib64` 多数指向 `/usr` 下对应目录 |
| `/var`   | 经常变化的数据，例如日志、缓存、队列、数据库文件                                                 |

## 命令格式

```bash
命令 [选项] [参数]
```

- `[]` 表示可选，不是实际输入内容。
- 命令、选项、参数都区分大小写。
- 回车后命令开始执行。
- 长命令可以用 `\` 换行。

```bash
systemctl list-unit-files | grep \
enable

systemctl list-unit-files | grep enable
```

这两种写法效果相同。

## 开关机命令

```bash
shutdown -h now     # 立即关机
shutdown -r now     # 立即重启
reboot now          # 立即重启
```

| 选项    | 含义        |
| ----- | --------- |
| `-h`  | halt，关机   |
| `-r`  | reboot，重启 |
| `now` | 立即执行      |

## 目录与文件命令

### ls

```bash
ls
ls -al
ls -alh
ls -alh --time-style=long-iso
ls -i
```

| 选项                      | 含义                  |
| ----------------------- | ------------------- |
| `-a`                    | 显示所有文件，包括隐藏文件       |
| `-l`                    | 长格式显示               |
| `-h`                    | 人类可读大小，通常和 `-l` 一起用 |
| `-i`                    | 显示 inode 编号         |
| `--time-style=long-iso` | 让时间显示更规整            |

### pwd

```bash
pwd
```

显示当前目录，例如：

```text
/root
```

### tree

```bash
tree
tree -C
tree -d
tree -u
```

| 选项   | 含义    |
| ---- | ----- |
| `-C` | 彩色显示  |
| `-d` | 只显示目录 |
| `-u` | 显示所有者 |

### cd

```bash
cd ..
cd ../..
cd -
cd ~
cd
cd /etc
```

| 写法            | 含义        |
| ------------- | --------- |
| `cd ..`       | 返回上一级目录   |
| `cd ../..`    | 返回上两级目录   |
| `cd -`        | 返回上一次所在目录 |
| `cd ~` / `cd` | 返回当前用户家目录 |
| `cd 目录`       | 切换到指定目录   |

### 绝对路径与相对路径

```bash
/etc/systemd/system.conf       # 绝对路径，从 / 开始
systemd/system.conf            # 相对路径，从当前目录开始
```

### touch

```bash
touch hello.txt
touch tomcat.txt nginx.txt redis.txt
```

`touch` 常用于创建空文件，也会更新已有文件的时间戳。

### mkdir

```bash
mkdir soft
mkdir a b c d
mkdir -p a/b/c
mkdir -pv a/b/c
```

| 选项   | 含义       |
| ---- | -------- |
| `-p` | 递归创建多级目录 |
| `-v` | 显示创建过程   |

### rm

```bash
rm 文件
rm -f 文件
rm -rf 目录
rm -rf *
```

> **注意**\
> `rm -rf` 删除后通常无法直接恢复。生产环境删除前应确认路径、确认备份、避免在变量为空时拼接出危险路径。

| 选项   | 含义                              |
| ---- | ------------------------------- |
| `-f` | 强制删除，不提示                        |
| `-r` | 递归删除目录                          |
| `*`  | Shell 通配符，表示当前目录下匹配到的所有非隐藏文件/目录 |

### mv

```bash
mv hello /app/
mv aaa bbb ccc /app/
mv aaa kkk
mv hello /app/logs/
mv han/* app/
```

| 场景      | 写法                |
| ------- | ----------------- |
| 移动文件    | `mv 文件 目标目录`      |
| 移动多个文件  | `mv 文件1 文件2 目标目录` |
| 重命名     | `mv 原名 新名`        |
| 只移动目录内容 | `mv 目录/* 目标目录/`   |

> **注意**\
> 目标位置存在同名文件或目录时，`mv` 可能覆盖原内容。

### cp

```bash
cp -a hello come
cp -a hello /app/
cp -a hello /app/come
cp -a aaa bbb ccc /app/
cp -a hello /app/
cp -a hello/* /app/
```

| 选项   | 含义                       |
| ---- | ------------------------ |
| `-r` | 递归复制目录，但可能改变属性和时间        |
| `-a` | 归档复制，递归并尽量保留原属性、时间、权限，常用 |

> **注意**\
> 目标位置存在同名文件或目录时，`cp` 也可能覆盖原内容。

## 常用快捷键

| 快捷键      | 作用         |
| -------- | ---------- |
| `Tab`    | 命令、文件、目录补全 |
| `Ctrl+a` | 光标到行首      |
| `Ctrl+e` | 光标到行尾      |
| `Ctrl+l` | 清屏         |
| `Ctrl+c` | 结束当前命令     |
| ↑ / ↓    | 调出历史命令     |

## 查看文件内容

### cat

```bash
cat 文件
cat -n 文件
```

`-n` 显示行号。

### head

```bash
head 文件
head -n 20 文件
```

默认显示前 10 行。

### tail

```bash
tail 文件
tail -n 20 文件
tail -f 文件
```

| 选项      | 含义         |
| ------- | ---------- |
| `-n 行数` | 显示最后多少行    |
| `-f`    | 实时跟踪文件新增内容 |

### man

```bash
man ls
man systemctl
```

查看系统命令、配置文件或系统调用帮助。

### more 与 less

`more` 会分页显示文件内容，`less` 更常用，支持前后滚动和搜索。

```bash
less 文件
less -N 文件
```

| less 操作   | 含义      |
| --------- | ------- |
| 空格        | 向下翻页    |
| `b`       | 向上翻页    |
| ↑ / ↓     | 上下移动    |
| `q`       | 退出      |
| `/字符串`    | 向后搜索    |
| `?字符串`    | 向前搜索    |
| `n`       | 继续同方向查找 |
| `Shift+n` | 反方向查找   |
| `-N`      | 显示行号    |

## 时间与时区

### timedatectl

```bash
timedatectl
```

查看系统时间、时区、NTP 状态。

### date

```bash
date
date -u
date "+%Y-%m-%d"
date "+%Y--%m--%d--%H:%M:%S"
```

| 格式   | 含义        |
| ---- | --------- |
| `%Y` | 四位年份      |
| `%y` | 两位年份      |
| `%m` | 月         |
| `%d` | 日         |
| `%H` | 24 小时制小时  |
| `%I` | 12 小时制小时  |
| `%M` | 分钟        |
| `%S` | 秒         |
| `-u` | 显示 UTC 时间 |

## history

```bash
history
history 20
```

查看历史命令，`history 20` 只显示最近 20 条。

## 重定向与标准输入输出

### 覆盖输出 `>`

```bash
ls > a.txt
> a.txt
```

- 文件不存在会自动创建。
- 文件存在会覆盖原内容。
- `> 文件名` 可以清空文件。

### 追加输出 `>>`

```bash
ls >> b.txt
```

- 文件不存在会自动创建。
- 文件存在时追加到末尾，不覆盖原内容。

### 标准流

| 编号  | 名称     | 含义   | 常用符号       |
| --- | ------ | ---- | ---------- |
| `0` | stdin  | 标准输入 | `<`、`<<`   |
| `1` | stdout | 标准输出 | `>`、`>>`   |
| `2` | stderr | 标准错误 | `2>`、`2>>` |

```bash
命令 > out.log          # 标准输出覆盖写入
命令 >> out.log         # 标准输出追加写入
命令 2> err.log         # 标准错误覆盖写入
命令 2>> err.log        # 标准错误追加写入
命令 > all.log 2>&1     # 标准输出和标准错误写到同一文件
```

`<<` 是 Here Document，表示把后续多行内容作为标准输入传给命令，不是普通追加写入。

## Vim 编辑器

### 三种模式

| 模式   | 说明                      |
| ---- | ----------------------- |
| 普通模式 | 打开文件后的默认模式，可移动、复制、删除、粘贴 |
| 编辑模式 | 输入文本                    |
| 命令模式 | 保存、退出、搜索、替换、设置行号        |

进入编辑模式：

```text
i I a A o O
```

回到普通模式：

```text
Esc
```

进入命令模式：

```text
: / ?
```

### 普通模式移动

| 操作          | 含义       |
| ----------- | -------- |
| `G`         | 到最后一行    |
| `gg`        | 到第一行     |
| `0`         | 到行首      |
| `$`         | 到行尾      |
| `n + Enter` | 向下移动 n 行 |
| `ngg`       | 到第 n 行   |

### 复制、粘贴、删除

| 操作    | 含义           |
| ----- | ------------ |
| `yy`  | 复制当前行        |
| `nyy` | 从当前行开始复制 n 行 |
| `p`   | 粘贴到下一行       |
| `P`   | 粘贴到上一行       |
| `dd`  | 删除当前行        |
| `ndd` | 从当前行开始删除 n 行 |
| `u`   | 撤销           |
| `x`   | 向后删除字符       |
| `X`   | 向前删除字符       |
| `.`   | 重复上一次操作      |
| `d1G` | 删除当前行到第一行    |
| `dG`  | 删除当前行到最后一行   |
| `d0`  | 删除当前光标到行首    |
| `d$`  | 删除当前光标到行尾    |

### 编辑模式入口

| 操作  | 含义            |
| --- | ------------- |
| `i` | 当前光标处插入       |
| `a` | 当前光标后插入       |
| `I` | 当前行第一个非空字符前插入 |
| `A` | 当前行末尾插入       |
| `O` | 当前行上一行新建      |
| `o` | 当前行下一行新建      |

### 命令模式

```vim
:wq
:q
:q!
:wq!
:set nu
:set nonu
```

| 命令          | 含义                                    |
| ----------- | ------------------------------------- |
| `:wq`       | 保存退出                                  |
| `:q`        | 未修改时退出                                |
| `:q!`       | 强制退出不保存                               |
| `:wq!`      | 强制写入并退出，但不等于自动提权。是否能写入仍取决于当前用户权限和文件权限 |
| `:set nu`   | 显示行号                                  |
| `:set nonu` | 取消行号                                  |

### 搜索与替换

```vim
/字符串
?字符串
n
N
:g/A/s//B/g
```

| 命令            | 含义                     |
| ------------- | ---------------------- |
| `/内容`         | 向下搜索                   |
| `?内容`         | 向上搜索                   |
| `n`           | 重复上一次搜索方向              |
| `N`           | 反方向重复搜索                |
| `:g/A/s//B/g` | 对匹配 A 的行执行替换，把 A 替换为 B |

## alias 别名

```bash
alias
alias 别名='命令'
unalias 别名
```

系统常见会给 `cp`、`mv`、`rm` 加上 `-i`，降低误操作风险。

查看别名：

```bash
alias
alias rm
```

设置别名：

```bash
alias rm='echo "不允许使用rm命令，谢谢"'
```

取消别名：

```bash
unalias rm
```

绕过别名：

```bash
/bin/rm 文件
\rm 文件
```

## file 查看文件类型

```bash
file /bin/cat
file /var/log/wtmp
file /dev/cdrom
file /dev/log
file /dev/sr0
file /dev/tty18
```

| 类型输出                        | 含义              |
| --------------------------- | --------------- |
| `ASCII text`                | 纯文本文件           |
| `ELF 64-bit LSB executable` | 二进制可执行文件        |
| `data`                      | 数据文件            |
| `directory`                 | 目录              |
| `symbolic link`             | 符号链接            |
| `socket`                    | 套接字文件           |
| `block special`             | 块设备，例如磁盘、光驱、U 盘 |
| `character special`         | 字符设备，例如终端、键盘、鼠标 |
| `pipe`                      | 管道文件            |

## ls -l 文件属性

```bash
ls -l /
```

典型输出列：

| 列       | 含义      |
| ------- | ------- |
| 第 1 列   | 文件类型和权限 |
| 第 2 列   | 硬链接数    |
| 第 3 列   | 属主      |
| 第 4 列   | 属组      |
| 第 5 列   | 文件大小    |
| 第 6-8 列 | 文件时间    |
| 第 9 列   | 文件名     |

## stat 与文件三种时间

```bash
stat test.txt
ls -l --time=ctime 文件
ls -l --time=atime 文件
```

| 时间             | 含义                          |
| -------------- | --------------------------- |
| Access / atime | 最近访问时间                      |
| Modify / mtime | 文件内容最近修改时间                  |
| Change / ctime | 文件元数据最近变更时间，例如权限、属主、大小、内容变化 |

`ls -l` 默认显示 `mtime`。`atime` 是否频繁更新还受挂载参数影响，例如 `relatime`、`noatime`。

## which 与 whereis

### which

```bash
which cp
which rm
which cd
```

`which` 根据 `$PATH` 查找命令路径。如果命令有别名，也可能同时显示别名信息。

### whereis

```bash
whereis cd
whereis ls
whereis -b ls
```

`whereis` 可查看命令路径和帮助文档位置，`-b` 只看二进制位置。

## locate

```bash
locate 文件名
updatedb
```

- `locate` 基于数据库 `/var/lib/mlocate/mlocate.db` 查找，速度快。
- 数据库不是实时更新，新增文件可能要等数据库更新后才能查到。
- `/tmp` 等临时目录通常可能不被记录。
- `updatedb` 会刷新数据库，但可能消耗 I/O，生产环境慎用。

## find

### 基本格式

```bash
find 要查找的目录 查找条件 查找到后的动作
```

`find` 直接遍历文件系统，范围越大越消耗资源。生产环境不要轻易从 `/` 开始全盘查找。

### 常用条件

| 条件              | 含义                                |
| --------------- | --------------------------------- |
| `-name`         | 按文件名查找                            |
| `-perm`         | 按权限查找                             |
| `-user`         | 按属主查找                             |
| `-group`        | 按属组查找                             |
| `-mtime -n/+n`  | 按修改时间查找，`-n` 为 n 天以内，`+n` 为 n 天以前 |
| `-type`         | 按文件类型查找                           |
| `-size`         | 按大小查找                             |
| `-maxdepth num` | 限制递归深度，应尽量写在其他条件前                 |

### 文件类型

| 类型  | 含义        |
| --- | --------- |
| `b` | 块设备       |
| `c` | 字符设备      |
| `d` | 目录        |
| `f` | 普通文件      |
| `l` | 符号链接      |
| `p` | 管道文件      |
| `s` | socket 文件 |

### size 注意点

```bash
find . -size +1M
find . -size -1024c
find . -size 1024c
```

- `+` 表示大于。
- `-` 表示小于。
- 不加 `+/-` 表示匹配对应大小单位范围。
- 要精确找 1KB，建议写 `1024c`，而不是 `1k`。

### 查找到后的动作

| 动作               | 含义                 |
| ---------------- | ------------------ |
| `-print`         | 输出匹配结果，默认动作        |
| `-exec 命令 {} \;` | 对每个匹配结果执行命令        |
| `-ok 命令 {} \;`   | 类似 `-exec`，但执行前会确认 |

### 示例

```bash
find . -name file.txt
find . -name "*.c"
find . -type f
find /home -size +1M
find /var/log -mtime +7
find /path/to/search -atime -7
find . -ctime 20
find . -ctime +20
find /var/log -type f -mtime +7 -ok rm {} \;
find . -type f -perm 644 -exec ls -l {} \;
find / -type f -size 0 -exec ls -l {} \;
find /path/to/search -name "pattern" -exec rm {} \;
```

## 命令速查

| 目标          | 命令                |
| ----------- | ----------------- |
| 当前路径        | `pwd`             |
| 长格式显示       | `ls -l`           |
| 显示隐藏文件      | `ls -a`           |
| 创建空文件       | `touch file`      |
| 创建目录        | `mkdir dir`       |
| 递归创建目录      | `mkdir -p a/b/c`  |
| 删除目录        | `rm -rf dir`      |
| 移动/重命名      | `mv src dst`      |
| 复制并保留属性     | `cp -a src dst`   |
| 查看文件        | `cat file`        |
| 分页查看        | `less file`       |
| 查看尾部        | `tail -n 20 file` |
| 实时跟踪        | `tail -f file`    |
| 查看类型        | `file file`       |
| 查看时间和 inode | `stat file`       |
| 查命令路径       | `which cmd`       |
| 查文件位置       | `find 路径 条件`      |

## 易错点

| 场景                 | 处理方式                   |
| ------------------ | ---------------------- |
| 把 `/tmp` 当长期存储目录   | 不要保存重要文件，可能被清理         |
| `rm -rf *` 路径没确认   | 执行前先 `pwd`、`ls` 确认当前目录 |
| `>` 覆盖文件           | 追加用 `>>`，覆盖前确认备份       |
| `cp` / `mv` 覆盖同名文件 | 先检查目标目录                |
| `find /` 全盘查找      | 尽量缩小路径范围               |
| `:wq!` 理解成自动提权     | Vim 不会自动获得 root 权限     |

## 关联条目

- [[Linux 文件系统]]
- [[Shell 重定向]]
- [[Vim]]
- [[find 命令]]
- [[Linux 文件属性]]
- [[CentOS 7]]
