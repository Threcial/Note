---
title: 网络命令、systemd 服务与进程管理
type: knowledge
status: evergreen
source: day07.md
created: 2026-06-13
updated: 2026-06-13
tags:
  - Linux
  - CentOS7
  - 网络排错
  - TCP
  - systemd
  - 进程管理
  - 服务管理
aliases:
  - DAY07
  - Linux 网络命令
  - TCP 连接状态
  - systemctl
  - Linux 进程管理
Belongs to:
  - "[[Linux 运维基础]]"
  - "[[网络排错]]"
  - "[[服务管理]]"
  - "[[进程管理]]"
Related to:
  - "[[DAY06 - RPM、YUM、源码安装与网络基础]]"
  - "[[IP 地址与子网掩码]]"
  - "[[静态路由]]"
  - "[[NAT]]"
  - "[[Linux 文件描述符]]"
  - "[[crond 定时任务]]"
Has:
  - "[[TCP 三次握手]]"
  - "[[TCP 四次挥手]]"
  - "[[netstat]]"
  - "[[ss]]"
  - "[[wget]]"
  - "[[curl]]"
  - "[[ip 命令]]"
  - "[[ifconfig]]"
  - "[[ping]]"
  - "[[telnet]]"
  - "[[nc]]"
  - "[[nslookup]]"
  - "[[dig]]"
  - "[[traceroute]]"
  - "[[systemctl]]"
  - "[[ps]]"
  - "[[pstree]]"
  - "[[top]]"
  - "[[kill]]"
  - "[[lsof]]"
_organized: true
---
# 网络命令、systemd 服务与进程管理

## 概览

| 主题   | 关键关系                                                    |
| ---- | ------------------------------------------------------- |
| 网络连接 | IP 决定主机定位，端口决定进程入口，TCP 状态反映连接生命周期                       |
| 监听排查 | 先看端口是否监听，再看进程，再看防火墙和路由                                  |
| 服务管理 | CentOS 7 以 `systemd` 为主，`systemctl` 是服务控制入口             |
| 进程管理 | 服务最终体现为进程，排错时经常从 `systemctl status` 追到 `ps/top/lsof/ss` |

## 网络排错链路

| 层次  | 关注点                | 常用命令                                      |
| --- | ------------------ | ----------------------------------------- |
| 网卡  | 网卡是否存在、是否 UP、是否有地址 | `ip addr`、`ip link`、`nmcli device`        |
| 路由  | 默认网关、目标网段路径        | `ip route`、`route -n`                     |
| 连通性 | ICMP 是否通、丢包、延迟     | `ping`、`traceroute`、`tracepath`           |
| DNS | 域名是否能解析            | `nslookup`、`dig`                          |
| 端口  | 服务端口是否监听、远端端口是否可达  | `ss`、`netstat`、`nc`、`telnet`              |
| 进程  | 哪个进程占用端口           | `ss -lntp`、`lsof -i:端口`、`ps`              |
| 服务  | 服务是否启动、是否开机自启      | `systemctl status`、`systemctl is-enabled` |

## 旧命令与新命令对应关系

| 旧命令                  | 更推荐的命令                 | 说明                                          |
| -------------------- | ---------------------- | ------------------------------------------- |
| `ifconfig`           | `ip addr`、`ip link`    | `ifconfig` 来自 `net-tools`，CentOS 7 默认可能没有安装 |
| `route -n`           | `ip route show`        | `ip route` 更完整，能覆盖路由查看、添加、删除                |
| `netstat -lntp`      | `ss -lntp`             | 大量连接场景下 `ss` 更快                             |
| `telnet ip port`     | `nc -vz ip port`       | 仅测试端口连通性时，`nc` 更直接                          |
| `service name start` | `systemctl start name` | CentOS 7 之后以 `systemctl` 为主                 |

## TCP 连接状态

TCP 通过序列号、确认号和标志位来维护连接状态。

| 字段    | 含义                  |
| ----- | ------------------- |
| `SYN` | 请求建立连接              |
| `seq` | 序列号，用于标识发送方的数据序列    |
| `ACK` | 确认标志                |
| `ack` | 确认号，通常是对方 `seq + 1` |

### 三次握手

| 步骤    | 客户端状态         | 服务端状态         | 报文核心                                 |
| ----- | ------------- | ------------- | ------------------------------------ |
| 服务端监听 | -             | `LISTEN`      | 服务端端口等待连接                            |
| 第一次握手 | `SYN-SENT`    | `LISTEN`      | 客户端发送 `SYN=1, seq=x`                 |
| 第二次握手 | `SYN-SENT`    | `SYN-RECV`    | 服务端回复 `SYN=1, ACK=1, seq=y, ack=x+1` |
| 第三次握手 | `ESTABLISHED` | `ESTABLISHED` | 客户端回复 `ACK=1, ack=y+1`               |

连接建立后，客户端和服务端都进入 `ESTABLISHED` 状态。

### 四次挥手

| 步骤  | 主动关闭方             | 被动关闭方             | 状态变化                                  |
| --- | ----------------- | ----------------- | ------------------------------------- |
| 第一次 | 发送 `FIN=1, seq=u` | 接收 FIN            | 主动方进入 `FIN_WAIT_1`                    |
| 第二次 | 接收 ACK            | 回复 `ACK, ack=u+1` | 被动方进入 `CLOSE_WAIT`，主动方进入 `FIN_WAIT_2` |
| 第三次 | 等待 FIN            | 发送 `FIN=1, ACK=1` | 被动方进入 `LAST_ACK`                      |
| 第四次 | 回复 ACK            | 接收 ACK            | 主动方进入 `TIME_WAIT`，被动方进入 `CLOSED`      |

`TIME_WAIT` 常出现在主动关闭连接的一方，作用是保证最后一个 ACK 有机会被对端收到，同时避免旧连接中的残留报文影响新连接。

## TCP 状态速查

| 状态            | 含义                   | 排查意义                 |
| ------------- | -------------------- | -------------------- |
| `LISTEN`      | 服务端正在监听端口            | 服务是否对外提供入口           |
| `SYN_SEND`    | 已发送 SYN，等待对方响应       | 常见于远端不可达、防火墙拦截、服务未监听 |
| `SYN_RECV`    | 收到 SYN，等待客户端确认       | 大量出现可能与半连接、攻击或网络异常有关 |
| `ESTABLISHED` | 连接已建立                | 正常通信状态               |
| `FIN_WAIT_1`  | 主动关闭方已发送 FIN         | 连接关闭过程               |
| `FIN_WAIT_2`  | 主动关闭方收到 ACK，等待对方 FIN | 对端迟迟不关闭时可能堆积         |
| `TIME_WAIT`   | 主动关闭方等待连接彻底过期        | 大量短连接服务常见            |
| `CLOSE_WAIT`  | 被动关闭方收到 FIN，但应用未关闭连接 | 大量出现通常是应用没有正确释放连接    |
| `LAST_ACK`    | 被动关闭方等待最后 ACK        | 连接关闭尾声               |
| `CLOSED`      | 连接关闭                 | 端口未使用或连接结束           |

## netstat 与 ss

### netstat

`netstat` 用于查看网络连接、监听端口、路由和接口信息。CentOS 7 需要安装 `net-tools`。

```bash
yum install net-tools -y
```

常用参数：

| 参数   | 含义               |
| ---- | ---------------- |
| `-i` | 显示网络接口信息         |
| `-a` | 显示全部连接和端口        |
| `-t` | TCP              |
| `-u` | UDP              |
| `-l` | 只显示监听状态          |
| `-p` | 显示进程名和 PID       |
| `-n` | 不解析服务名和域名，直接显示数字 |

常用命令：

```bash
netstat -a
netstat -natp
netstat -lntp
netstat -lt
```

输出字段：

| 字段                 | 含义              |
| ------------------ | --------------- |
| `Proto`            | 协议              |
| `Recv-Q`           | 接收队列，非 0 可能表示阻塞 |
| `Send-Q`           | 发送队列，非 0 可能表示阻塞 |
| `Local Address`    | 本地监听地址和端口       |
| `Foreign Address`  | 对端地址和端口         |
| `State`            | TCP 状态          |
| `PID/Program name` | 进程 ID 和进程名      |

### ss

`ss` 是更推荐的连接查看命令。连接数很大时，`ss` 通常比 `netstat` 更快。

```bash
ss -lntp
ss -antp
ss -atpn
ss -lunp
```

| 命令         | 用途               |
| ---------- | ---------------- |
| `ss -lntp` | 查看 TCP 监听端口和对应进程 |
| `ss -antp` | 查看所有 TCP 连接      |
| `ss -lunp` | 查看 UDP 监听端口      |
| `ss -s`    | 查看连接统计概要         |

## 下载与 HTTP 测试

### wget

`wget` 更偏向下载文件，默认下载到当前目录，并支持断点续传。

| 参数                 | 含义        |
| ------------------ | --------- |
| `-q`               | 安静模式      |
| `-nv`              | 关闭无用提示    |
| `-O`               | 指定下载后的文件名 |
| `-b`               | 后台下载      |
| `-c`               | 断点续传      |
| `--limit-rate=10k` | 限速下载      |

```bash
wget http://mirrors.aliyun.com/centos/7/isos/x86_64/CentOS-7-x86_64-Everything-2009.torrent
wget -O eving.torrent http://mirrors.aliyun.com/centos/7/isos/x86_64/CentOS-7-x86_64-Everything-2009.torrent
wget -b -O js.torrent http://mirrors.aliyun.com/centos/7/isos/x86_64/CentOS-7-x86_64-Everything-2009.torrent
wget -c --limit-rate=1m URL
```

### curl

`curl` 更适合测试 URL、查看响应头、调用 HTTP 接口，也可以下载文件。

| 参数   | 含义        |
| ---- | --------- |
| `-o` | 指定输出文件名   |
| `-O` | 使用远端文件名保存 |
| `-I` | 只查看响应头    |
| `-L` | 跟随重定向     |
| `-s` | 静默输出      |
| `-w` | 自定义输出统计信息 |

```bash
curl www.baidu.com
curl -I https://www.bilibili.com/
curl -o eything.torrent http://mirrors.aliyun.com/centos/7/isos/x86_64/CentOS-7-x86_64-Everything-2009.torrent
curl -O http://mirrors.aliyun.com/centos/7/isos/x86_64/CentOS-7-x86_64-Everything-2009.torrent
```

常用 HTTP 连通性测试：

```bash
curl -I -L https://example.com
curl -s -o /dev/null -w "%{http_code} %{time_total}\n" https://example.com
```

## 网络配置

### 查看网卡

```bash
ip addr
ip a
ip addr show dev ens33
ip link show
ip neigh
```

`ip neigh` 用于查看 ARP 邻居表。

旧命令：

```bash
ifconfig
```

如果没有 `ifconfig`：

```bash
yum install net-tools -y
```

### 临时添加 IP

原始方式：

```bash
ifconfig ens33:1 192.168.8.5 netmask 255.255.255.0 up
```

更推荐使用 `ip addr`：

```bash
ip addr add 192.168.8.5/24 dev ens33
ip addr del 192.168.8.5/24 dev ens33
```

### 修改 CentOS 7 网卡配置

配置文件示例：

```ini
BOOTPROTO=static
IPADDR=192.168.0.168
NETMASK=255.255.255.0
GATEWAY=192.168.0.1
DNS1=192.168.0.1
ONBOOT=yes
```

文件路径通常是：

```bash
/etc/sysconfig/network-scripts/ifcfg-ens33
```

重启网卡：

```bash
ifdown ens33
ifup ens33
```

也可以使用 NetworkManager：

```bash
nmcli device status
nmcli connection show
nmcli connection reload
nmcli connection up ens33
```

## 路由查看与配置

查看路由表：

```bash
route -n
ip route show
ip route get 8.8.8.8
```

添加临时静态路由：

```bash
route add -net 22.22.22.0 netmask 255.255.255.0 gw 192.168.8.1
```

对应的 `ip route` 写法：

```bash
ip route add 22.22.22.0/24 via 192.168.8.1
ip route del 22.22.22.0/24 via 192.168.8.1
```

CentOS 7 永久静态路由可以写入网卡路由文件：

```bash
vim /etc/sysconfig/network-scripts/route-eth0
```

内容示例：

```text
10.15.150.0/24 via 192.168.8.1 dev eth0
10.25.250.0/24 via 192.168.8.1 dev eth0
```

重启网络：

```bash
systemctl restart network
```

## 探测命令

| 命令               | 用途                 |
| ---------------- | ------------------ |
| `ping`           | 测试 ICMP 连通性        |
| `telnet ip port` | 测试 TCP 端口是否可连接     |
| `nc -vz ip port` | 更推荐的端口探测方式         |
| `nslookup 域名`    | 查询域名解析             |
| `dig 域名`         | 更详细的 DNS 查询        |
| `traceroute IP`  | 查看到目标经过的路由节点       |
| `tracepath IP`   | 不依赖 root 权限的路径探测方式 |

示例：

```bash
ping -c 4 192.168.1.1
ping -s 1400 -c 4 192.168.1.1
telnet 192.168.1.10 80
nc -vz 192.168.1.10 80
nslookup www.baidu.com
dig www.baidu.com
traceroute 8.8.8.8
```

## systemd 服务管理

### 服务概念

Linux 中的服务是持续在后台运行、等待用户或其他程序调用的程序。服务分为系统服务和网络服务：

| 类型   | 服务对象      |
| ---- | --------- |
| 系统服务 | 本机系统和本机用户 |
| 网络服务 | 网络中的其他客户端 |

### init 与 systemd

CentOS 6 常用 System V init：

```bash
/etc/init.d/servername start
service servername start
chkconfig --list servername
chkconfig --level 3 servername on
```

CentOS 7 开始以 `systemd` 为主，`systemctl` 是主要管理入口。`service` 在 CentOS 7 中通常只是兼容封装。

### 运行级别与 target

| 传统运行级别 | systemd target      | 含义       |
| ------ | ------------------- | -------- |
| 0      | `poweroff.target`   | 关机       |
| 1      | `rescue.target`     | 单用户/救援模式 |
| 3      | `multi-user.target` | 多用户字符模式  |
| 5      | `graphical.target`  | 图形模式     |
| 6      | `reboot.target`     | 重启       |

查看默认 target：

```bash
systemctl get-default
```

设置默认 target：

```bash
systemctl set-default multi-user.target
```

### systemd 单元文件目录

| 路径                         | 含义             | 优先级 |
| -------------------------- | -------------- | --- |
| `/usr/lib/systemd/system/` | 软件包安装的服务单元     | 低   |
| `/run/systemd/system/`     | 系统运行时生成的服务单元   | 中   |
| `/etc/systemd/system/`     | 管理员自定义或覆盖的服务单元 | 高   |

### systemctl 常用动作

```bash
systemctl start httpd
systemctl stop httpd
systemctl restart httpd
systemctl reload httpd
systemctl status httpd
systemctl is-active httpd
systemctl enable httpd
systemctl disable httpd
systemctl is-enabled httpd
```

| 动作           | 含义           |
| ------------ | ------------ |
| `start`      | 启动服务         |
| `stop`       | 停止服务         |
| `restart`    | 重启服务，未启动也会启动 |
| `reload`     | 重新加载配置文件     |
| `status`     | 查看状态         |
| `is-active`  | 判断是否运行       |
| `enable`     | 设置开机自启       |
| `disable`    | 取消开机自启       |
| `is-enabled` | 判断是否开机自启     |

修改 unit 文件后需要重新加载 systemd：

```bash
systemctl daemon-reload
```

### systemctl status 关键字段

| 字段         | 含义                |
| ---------- | ----------------- |
| `Loaded`   | 服务单元是否加载，以及是否开机启动 |
| `Active`   | 当前运行状态            |
| `Main PID` | 主进程号              |
| `Tasks`    | 进程和线程数量           |
| `Memory`   | 当前占用内存            |
| `CGroup`   | 该服务的控制组           |
| 日志行        | 最近的服务启动、停止或错误日志   |

常见状态：

| 状态                 | 含义              |
| ------------------ | --------------- |
| `active (running)` | 正在运行            |
| `active (exited)`  | 执行过并退出，常见于一次性服务 |
| `active (waiting)` | 正在等待其他事件        |
| `inactive (dead)`  | 未运行             |
| `activating`       | 启动中             |
| `deactivating`     | 停止中             |
| `failed`           | 启动失败            |

查看完整日志：

```bash
journalctl -u httpd
journalctl -u httpd -f
journalctl -xe
```

## 进程管理

### 进程与程序

| 概念 | 说明                           |
| -- | ---------------------------- |
| 程序 | 静态指令和资源，存放在磁盘中               |
| 进程 | 程序运行后产生的活动实体，有自己的 PID 和资源上下文 |

一个程序可以启动多个进程，一个进程也可能再派生子进程。进程之间存在父子关系，父进程 PID 记为 PPID。

## ps

`ps` 是静态进程查看命令，显示执行命令那一刻的进程状态。

常用命令：

```bash
ps -ef
ps aux
ps -l
ps -u root
```

常用参数：

| 参数   | 含义                 |
| ---- | ------------------ |
| `-A` | 所有进程               |
| `-a` | 与终端有关的进程           |
| `-x` | 常与 `-a` 组合，显示无终端进程 |
| `-u` | 指定用户               |
| `-l` | 长格式                |
| `-f` | 完整格式               |

`ps aux` 字段：

| 字段        | 含义            |
| --------- | ------------- |
| `USER`    | 启动进程的用户       |
| `PID`     | 进程号           |
| `%CPU`    | CPU 占用率       |
| `%MEM`    | 物理内存占用率       |
| `VSZ`     | 虚拟内存大小        |
| `RSS`     | 物理内存占用        |
| `TTY`     | 终端，`?` 表示系统进程 |
| `STAT`    | 进程状态          |
| `START`   | 启动时间          |
| `TIME`    | 消耗的 CPU 时间    |
| `COMMAND` | 命令            |

`ps -l` 常见状态：

| 状态  | 含义                 |
| --- | ------------------ |
| `R` | 运行中                |
| `S` | 可唤醒睡眠              |
| `D` | 不可中断睡眠，常与 I/O 等待有关 |
| `T` | 停止                 |
| `Z` | 僵尸进程               |

## pstree

```bash
pstree -p
pstree -u nobody
```

| 参数   | 含义     |
| ---- | ------ |
| `-p` | 显示 PID |
| `-u` | 显示所属用户 |

`pstree` 适合观察父子进程关系，例如 `nginx` master/worker 结构。

## top

`top` 用于动态观察系统资源和进程。

```bash
top
top -d 1
top -n 3 -d 1
top -p 7726
```

| 参数   | 含义        |
| ---- | --------- |
| `-d` | 指定刷新间隔    |
| `-n` | 刷新指定次数后退出 |
| `-p` | 查看指定 PID  |

顶部信息：

| 区域        | 含义                          |
| --------- | --------------------------- |
| 第一行       | 当前时间、运行时长、登录用户数、1/5/15 分钟负载 |
| `Tasks`   | 进程数量统计                      |
| `%Cpu(s)` | CPU 使用情况                    |
| `Mem`     | 内存使用情况                      |
| `Swap`    | swap 使用情况                   |

CPU 字段：

| 字段   | 含义            |
| ---- | ------------- |
| `us` | 用户态 CPU       |
| `sy` | 内核态 CPU       |
| `ni` | 调整过优先级的进程 CPU |
| `id` | 空闲 CPU        |
| `wa` | I/O 等待        |
| `hi` | 硬中断           |
| `si` | 软中断           |
| `st` | 虚拟化偷取时间       |

交互按键：

| 按键  | 作用         |
| --- | ---------- |
| `1` | 显示所有 CPU 核 |
| `M` | 按内存排序      |
| `P` | 按 CPU 排序   |
| `N` | 按 PID 排序   |
| `R` | 反向排序       |
| `T` | 按 CPU 时间排序 |
| `q` | 退出         |

## pidof、pgrep、kill、killall

查看进程 PID：

```bash
pidof sshd
pgrep sshd
pgrep -a sshd
```

`kill` 通过信号控制进程：

```bash
kill PID
kill -15 PID
kill -9 PID
kill -1 PID
kill -l
```

常见信号：

| 信号    | 含义                      |
| ----- | ----------------------- |
| `-1`  | 重新加载配置，类似部分服务的 `reload` |
| `-2`  | 中断，通常来自 Ctrl+C          |
| `-15` | 正常结束，默认信号               |
| `-9`  | 强制结束，无法被进程捕获，最后手段       |

按进程名结束：

```bash
killall ping
killall -i ping
killall -I ping
killall -e ping
```

| 参数   | 含义    |
| ---- | ----- |
| `-i` | 交互确认  |
| `-I` | 忽略大小写 |
| `-e` | 精确匹配  |

更现代的替代用法：

```bash
pkill ping
pkill -f "python app.py"
```

## lsof

`lsof` 用于查看进程打开的文件、端口和 socket。

```bash
lsof -i:22
lsof -u root
lsof -p 1
lsof /var/log/messages
```

| 参数     | 含义             |
| ------ | -------------- |
| `-i`   | 查看网络端口         |
| `-u`   | 查看指定用户打开的文件    |
| `-p`   | 查看指定 PID 打开的文件 |
| 直接跟文件名 | 查看哪些进程正在使用这个文件 |

常见场景：

```bash
lsof -iTCP:80 -sTCP:LISTEN
lsof +D /mnt/data
```

## 易错点

| 问题                                | 说明                              |
| --------------------------------- | ------------------------------- |
| `netstat` 没有命令                    | 需要安装 `net-tools`，但更推荐使用 `ss`    |
| `ifconfig` 没有命令                   | 同样属于 `net-tools`，推荐使用 `ip addr` |
| `TIME_WAIT` 很多不一定是故障              | 短连接服务常见，需结合端口耗尽、负载和业务情况判断       |
| `CLOSE_WAIT` 很多更值得关注              | 通常说明应用没有正确关闭连接                  |
| `systemctl restart` 与 `reload` 不同 | `restart` 重启进程，`reload` 只重新加载配置 |
| `kill -9` 不应作为默认手段                | 它不会给进程清理资源的机会                   |
