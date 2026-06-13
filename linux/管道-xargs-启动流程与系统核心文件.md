---
title: 管道、xargs、启动流程与系统核心文件
type: knowledge
status: evergreen
source: day02.md
created: 2026-06-13
updated: 2026-06-13
tags:
  - Linux
  - CentOS7
  - 管道
  - xargs
  - systemd
  - SELinux
  - Shell
aliases:
  - DAY02
  - Linux 管道与 xargs
  - CentOS7 启动流程
  - Linux 核心目录文件
Belongs to:
  - "[[Linux 运维基础]]"
  - "[[CentOS 7]]"
Related to:
  - "[[day01 CentOS 7 基础环境与常用命令]]"
  - "[[Linux 常用命令]]"
  - "[[find 命令]]"
  - "[[Shell 通配符]]"
  - "[[systemd]]"
Has:
  - "[[管道符]]"
  - "[[xargs]]"
  - "[[CentOS 7 启动流程]]"
  - "[[etc 目录核心文件]]"
  - "[[proc 目录]]"
  - "[[hostnamectl]]"
  - "[[SELinux]]"
_organized: true
---
# 管道、xargs、启动流程与系统核心文件

# 概览

- 管道 `|` 传递的是上一个命令的标准输出文本。
- `xargs` 用于把标准输入转换成后续命令的参数，经常和 `find` 搭配。
- CentOS 7 的启动核心是 `systemd`，默认目标通常是 `multi-user.target` 或 `graphical.target`。
- `/etc`、`/usr`、`/var`、`/proc` 是运维中最常接触的目录。
- SELinux 是内核级强制访问控制机制，实验环境常关闭，生产环境应按安全策略评估。
- Shell 通配符发生在命令执行前，由 Shell 先展开，再把结果交给命令。

## 关系表

| 条目          | 上游依赖                         | 下游用途             |
| ----------- | ---------------------------- | ---------------- |
| 管道符         |                              | 标准输出、标准错误        |
| `xargs`     | 标准输入、文件列表                    | 批量处理文件、把文本变成参数   |
| 启动流程        | BIOS/MBR/GRUB/kernel/systemd | 排查开机失败、服务启动顺序    |
| `/etc` 核心文件 | Linux 目录结构                   | 网络、DNS、挂载、系统参数配置 |
| SELinux     | 权限系统、内核安全模块                  | 服务访问异常、安全加固      |
| 通配符         | Shell 解析机制                   | 批量匹配文件名          |

## 管道符

```bash
ls -al /etc | less
```

管道符 `|` 的作用是把左边命令的标准输出交给右边命令处理。

```text
命令A 的输出文本 -> 命令B 的输入
```

管道传递的是文本流，不是“文件对象”。右侧命令必须能从标准输入读取数据，否则管道结果可能不符合预期。

## xargs

### 基本作用

有些命令不从标准输入读取目标，而是需要把目标写成命令参数。这时可以用 `xargs` 把输入文本转换成参数。

```bash
find /test -type f | xargs ls -l
```

对比：

```bash
find /test -type f | ls -l
```

这条通常不是想要的效果，因为 `ls` 不会把管道传入的文本当作要列出的文件参数。

### ls、cat 与 xargs 的差异

```bash
ls | cat
ls | xargs cat
```

| 命令   | 行为        |
| ---- | --------- |
| `ls` | cat       |
| `ls` | xargs cat |

## CentOS 7 启动流程

### 概览

```text
POST 开机自检
-> BIOS 选择启动设备
-> 读取 MBR 中的 Bootloader
-> 加载 GRUB 菜单
-> 加载 kernel
-> initramfs 临时根文件系统加载核心模块
-> 挂载真正的根目录
-> 启动 systemd
-> 进入 default.target
-> 执行 sysinit.target
-> 执行 basic.target
-> 执行 multi-user.target 或 graphical.target
-> 启动 tty / 图形桌面
```

### 传统运行级别与 systemd target

| 运行级别 | systemd target      | 含义             |
| ---- | ------------------- | -------------- |
| 0    | `poweroff.target`   | 关机             |
| 1    | `rescue.target`     | 单用户/救援模式       |
| 2    | `multi-user.target` | 多用户，传统上不含完整网络  |
| 3    | `multi-user.target` | 多用户文本模式，常用默认级别 |
| 4    | `multi-user.target` | 保留             |
| 5    | `graphical.target`  | 图形界面多用户模式      |
| 6    | `reboot.target`     | 重启             |

### rc.local

CentOS 7 中：

```text
/etc/rc.local -> /etc/rc.d/rc.local
```

`rc.local` 默认可能没有执行权限。该方式已不推荐作为长期方案。开机自启脚本更适合制作成 systemd service，再用 `systemctl enable` 加入开机启动。

## 用户登录时加载的文件

所有用户登录时会加载：

```bash
/etc/profile
/etc/profile.d/
```

用户自己的配置：

```bash
~/.bashrc
```

| 文件/目录             | 作用               |
| ----------------- | ---------------- |
| `/etc/profile`    | 全局环境变量           |
| `/etc/profile.d/` | 全局脚本目录           |
| `~/.bashrc`       | 当前用户自己的 Shell 配置 |

## /etc 核心文件

| 路径                                           | 作用                                |
| -------------------------------------------- | --------------------------------- |
| `/etc/sysconfig/network-scripts/ifcfg-ens33` | CentOS 7 网卡配置文件                   |
| `/etc/resolv.conf`                           | DNS 服务器配置，优先级通常低于网卡配置中的 DNS       |
| `/etc/hosts`                                 | 本地域名与 IP 映射，局域网解析常用               |
| `C:\Windows\System32\drivers\etc\HOSTS`      | Windows 对应 hosts 文件               |
| `/etc/fstab`                                 | 开机自动挂载配置，错误配置可能导致启动异常             |
| `/etc/rc.local`                              | 开机自启命令入口，链接到 `/etc/rc.d/rc.local` |
| `/etc/profile`                               | 全局变量                              |
| `/etc/bashrc`                                | 全局 Bash 配置                        |
| `/etc/issue`                                 | 登录前显示信息                           |
| `/etc/motd`                                  | 登录后欢迎/提示信息                        |
| `/etc/redhat-release`                        | 系统版本信息                            |
| `/etc/sysctl.conf`                           | 内核参数配置                            |
| `/etc/init.d`                                | CentOS 7 之前常用服务脚本目录               |

DNS 的作用是把域名解析成 IP 地址。

## /usr 核心目录

| 路径           | 作用                 |
| ------------ | ------------------ |
| `/usr/local` | 编译安装软件的默认位置        |
| `/usr/src`   | 源码存放路径             |
| `/usr/sbin`  | 开机时不一定需要、偏管理类命令和脚本 |
| `/usr/share` | 帮助文档和共享文件          |

## /var 核心目录

| 路径                  | 作用                |
| ------------------- | ----------------- |
| `/var/log`          | 系统和服务日志目录         |
| `/var/log/messages` | 系统级日志，记录正常与异常运行行为 |
| `/var/log/secure`   | 用户登录、安全认证相关日志     |
| `/var/log/dmesg`    | 硬件和内核启动信息         |

日志轮转是系统定期生成新日志文件、归档旧日志文件的机制。

## /proc 核心文件

`/proc` 是内存中的虚拟文件系统，反映内核、进程和硬件运行状态。

| 路径              | 作用     | 对应命令      |
| --------------- | ------ | --------- |
| `/proc/meminfo` | 内存信息   | `free -h` |
| `/proc/cpuinfo` | CPU 信息 | `lscpu`   |
| `/proc/loadavg` | 系统负载   | `uptime`  |

`uptime` 示例：

```yaml
10:59:37 up 23:07,  2 users,  load average: 0.00, 0.01, 0.05
```

| 字段             | 含义                  |
| -------------- | ------------------- |
| `10:59:37`     | 当前时间                |
| `up 23:07`     | 已运行时间               |
| `2 users`      | 当前登录用户数             |
| `load average` | 1 分钟、5 分钟、15 分钟平均负载 |

## 主机名查看与更改

主机名记录在：

```bash
/etc/hostname
```

查看：

```bash
cat /etc/hostname
hostname
hostnamectl
```

临时修改：

```bash
hostname test
```

重启后失效。

永久修改：

```bash
hostnamectl set-hostname didiyun
cat /etc/hostname
```

断开重连后提示符中的主机名通常会更新。

## SELinux

SELinux 是 Security-Enhanced Linux，属于内核安全模块，提供强制访问控制策略。

查看状态：

```bash
getenforce
```

临时关闭：

```bash
setenforce 0
```

永久关闭配置文件：

```bash
vim /etc/selinux/config
```

```ini
SELINUX=disabled
```

> **注意**\
> 生产环境不应简单把 SELinux 当成“必须关闭的东西”。如果服务异常与 SELinux 有关，优先确认策略、上下文和安全要求。

## Shell 通配符

通配符由 Shell 解析，用于匹配文件名。

### `*`

匹配任意字符，数量可以是 0 个或多个。

```bash
cp * /tmp/
ls *.txt
```

`*` 默认不匹配隐藏文件。

### `?`

匹配任意单个字符。

```bash
ls ?.txt
ls ??.txt
ls ????.txt
```

### `[]`

匹配中括号中的任意一个字符。

```bash
ls [a-z].txt
ls [ahdwk].txt
ls [abc].txt
```

`[test].txt` 表示匹配 `t.txt`、`e.txt`、`s.txt` 中的一个，并不表示匹配 `test.txt`。

### `[!...]` 与 `[^...]`

取反匹配。

```bash
ls [!ab].txt
ls [^ab].txt
```

## 其他 Shell 特殊符号

### `;`

命令分隔符，前一个命令失败不影响后一个命令继续执行。

```bash
echodate ; cd /root ; mkf9 123 ; touch jjjkkk
```

即使 `echodate` 和 `mkf9` 不存在，`cd /root` 和 `touch jjjkkk` 仍会执行。

### `\`

转义字符，让特殊字符还原字面含义。

```bash
echo $name
echo \$name
```

### `{起始..结束}`

生成序列。

```bash
echo {1..6}
echo {a..g}
```

### `&&` 与 `||`

| 符号     | 含义              |
| ------ | --------------- |
| `&&`   | 前一个命令成功才执行后一个命令 |
| `\|\|` | 前一个命令失败才执行后一个命令 |

```bash
mkdir 123456 2>/dev/null && echo '成功' || echo '失败'
```

## 命令速查

| 目标           | 命令                            |
| ------------ | ----------------------------- |
| 管道分页         | `ls -al /etc                  |
| find 结果交给 ls | `find /test -type f           |
| 查看启动目标       | `systemctl get-default`       |
| 修改主机名        | `hostnamectl set-hostname 名称` |
| 查看 SELinux   | `getenforce`                  |
| 临时关闭 SELinux | `setenforce 0`                |
| 查看负载         | `uptime`                      |
| 查看内存         | `free -h`                     |
| 查看 CPU       | `lscpu`                       |

## 易错点

| 场景                  | 说明                           |
| ------------------- | ---------------------------- |
| 把管道理解成传文件           | 管道传的是文本流，右侧命令要能处理标准输入        |
| `find ...           | ls -l`                       |
| `*` 匹配隐藏文件          | 默认不匹配隐藏文件                    |
| `[test]` 匹配字符串      | 实际只匹配其中任意一个字符                |
| 依赖 `rc.local` 开机自启  | CentOS 7 更推荐 systemd service |
| 修改 `/etc/fstab` 未验证 | 可能导致启动失败，改完应测试挂载             |

## 关联条目

- [[Shell 重定向]]
- [[find 命令]]
- [[xargs]]
- [[systemd]]
- [[SELinux]]
- [[Shell 通配符]]
- [[Linux 日志]]
