---
title: firewalld、防火墙区域、后台任务与远程传输
type: knowledge
status: evergreen
source: day09.md
created: 2026-06-13
updated: 2026-06-13
tags:
  - Linux
  - CentOS7
  - firewalld
  - firewall-cmd
  - nohup
  - SSH
  - rsync
  - 系统监控
aliases:
  - DAY09
  - firewalld 防火墙
  - Linux 后台任务
  - SSH 密钥登录
  - rsync 远程同步
Belongs to:
  - "[[Linux 运维基础]]"
  - "[[网络安全]]"
  - "[[远程运维]]"
  - "[[系统监控]]"
Related to:
  - "[[DAY07 - 网络命令、systemd 服务与进程管理]]"
  - "[[systemctl]]"
  - "[[ss]]"
  - "[[lsof]]"
  - "[[SSH]]"
  - "[[Shell 重定向]]"
  - "[[tar 打包压缩]]"
Has:
  - "[[firewalld]]"
  - "[[firewall-cmd]]"
  - "[[firewalld zone]]"
  - "[[rich rule]]"
  - "[[nohup]]"
  - "[[jobs]]"
  - "[[fg]]"
  - "[[uptime]]"
  - "[[free]]"
  - "[[lscpu]]"
  - "[[last]]"
  - "[[iftop]]"
  - "[[iostat]]"
  - "[[iotop]]"
  - "[[ssh-keygen]]"
  - "[[ssh-copy-id]]"
  - "[[rsync]]"
---
# firewalld、防火墙区域、后台任务与远程传输

## 概览

| 主题        | 结论                                                 |
| --------- | -------------------------------------------------- |
| firewalld | 基于 zone 管理流量，规则匹配时 source 优先于 interface            |
| 临时规则      | 不加 `--permanent` 的规则只在运行时生效，重启或 reload 后可能丢失       |
| 永久规则      | 加 `--permanent` 后需要 `firewall-cmd --reload` 才进入运行时 |
| 后台任务      | `&` 只负责后台运行，`nohup` 负责忽略 HUP 信号，两者不是一回事            |
| rsync     | 目录末尾 `/` 会改变同步语义：是否包含目录本身                          |

## firewalld 区域

CentOS 7 的 `firewalld` 引入了区域概念。不同区域代表不同信任级别。

| 区域         | 默认策略与用途                        |
| ---------- | ------------------------------ |
| `drop`     | 丢弃所有进入本机的包，不回复；本机仍可主动访问外部      |
| `block`    | 拒绝进入本机的包，并回复拒绝信息               |
| `public`   | 默认区域，通常只允许 SSH 和 dhcpv6-client |
| `external` | 面向外部网络，通常用于 NAT 或网关场景          |
| `dmz`      | 隔离区，只开放少量入口                    |
| `work`     | 工作网络，默认比 public 略宽             |
| `home`     | 家庭网络，默认允许 mdns、samba-client 等  |
| `internal` | 内部网络，默认允许内部常见服务                |
| `trusted`  | 接受所有连接，不做限制                    |

`drop` 和 `block` 的主要区别是：`drop` 不回复，`block` 会回复拒绝信息。

## firewall-cmd 常用参数

| 参数                             | 作用              |
| ------------------------------ | --------------- |
| `--get-default-zone`           | 查看默认区域          |
| `--set-default-zone=<zone>`    | 设置默认区域          |
| `--get-zones`                  | 查看可用区域          |
| `--get-services`               | 查看可用服务          |
| `--get-active-zones`           | 查看当前活跃区域        |
| `--zone=<zone>`                | 指定操作区域          |
| `--list-all`                   | 查看指定区域规则        |
| `--list-all-zones`             | 查看所有区域规则        |
| `--add-source=<IP/CIDR>`       | 把来源地址绑定到区域      |
| `--remove-source=<IP/CIDR>`    | 移除来源地址绑定        |
| `--add-interface=<网卡>`         | 把网卡加入区域         |
| `--change-interface=<网卡>`      | 修改网卡所属区域        |
| `--get-zone-of-interface=<网卡>` | 查看网卡所属区域        |
| `--add-port=80/tcp`            | 开放端口            |
| `--remove-port=80/tcp`         | 移除端口            |
| `--add-protocol=icmp`          | 放行协议            |
| `--remove-protocol=icmp`       | 移除协议            |
| `--add-rich-rule`              | 添加富规则           |
| `--remove-rich-rule`           | 删除富规则           |
| `--reload`                     | 重新加载规则          |
| `--panic-on`                   | 开启应急模式，断开所有网络连接 |
| `--panic-off`                  | 关闭应急模式          |

启动和停止服务：

```bash
systemctl start firewalld.service
systemctl restart firewalld.service
systemctl stop firewalld.service
systemctl status firewalld.service
```

## 运行时规则与永久规则

运行时规则：

```bash
firewall-cmd --zone=drop --add-port=80/tcp
```

永久规则：

```bash
firewall-cmd --permanent --zone=drop --add-port=80/tcp
firewall-cmd --reload
```

把当前运行时配置保存成永久配置：

```bash
firewall-cmd --runtime-to-permanent
```

查看当前规则：

```bash
firewall-cmd --get-active-zones
firewall-cmd --zone=trusted --list-all
firewall-cmd --zone=drop --list-rich-rules
```

## zone 匹配逻辑

firewalld 接收到请求后，区域匹配大致按下面顺序理解：

| 优先级 | 匹配依据         | 说明                                    |
| --- | ------------ | ------------------------------------- |
| 1   | source       | 如果来源地址被绑定到某个 zone，优先走 source 对应的 zone |
| 2   | interface    | 如果 source 没匹配上，再看流量进入的网卡属于哪个 zone     |
| 3   | default zone | 如果都没有明确匹配，则使用默认 zone                  |

示例：

```bash
firewall-cmd --zone=trusted --add-source=192.168.0.81
firewall-cmd --zone=drop --change-interface=ens33
```

此时：

| 来源               | 处理区域      |
| ---------------- | --------- |
| `192.168.0.81`   | `trusted` |
| 其他访问 `ens33` 的来源 | `drop`    |

一个来源地址只能绑定到一个区域。如果要改区域，先移除旧绑定：

```bash
firewall-cmd --zone=trusted --remove-source=192.168.0.81
firewall-cmd --zone=home --add-source=192.168.0.81
```

## 常用防火墙配置示例

### 把网卡切到 drop 区域

```bash
firewall-cmd --zone=drop --change-interface=ens33
```

永久生效：

```bash
firewall-cmd --permanent --zone=drop --change-interface=ens33
firewall-cmd --reload
```

### 在 drop 区域开放 80 端口

```bash
firewall-cmd --zone=drop --add-port=80/tcp
```

永久生效：

```bash
firewall-cmd --permanent --zone=drop --add-port=80/tcp
firewall-cmd --reload
```

### 只允许指定 IP 访问 22 端口

```bash
firewall-cmd --zone=drop --permanent --add-rich-rule='rule family="ipv4" source address="192.168.0.81" port protocol="tcp" port="22" accept'
firewall-cmd --reload
```

移除：

```bash
firewall-cmd --zone=drop --permanent --remove-rich-rule='rule family="ipv4" source address="192.168.0.81" port protocol="tcp" port="22" accept'
firewall-cmd --reload
```

### 允许指定 IP ping

```bash
firewall-cmd --zone=drop --add-rich-rule='rule family="ipv4" source address="192.168.31.77" protocol value="icmp" accept'
```

### 在 drop 区域允许或禁止 ICMP

```bash
firewall-cmd --zone=drop --add-protocol=icmp
firewall-cmd --zone=drop --remove-protocol=icmp
```

### 使用 service 方式开放服务

firewalld 内置了很多服务模板，例如 `ssh`、`http`、`https`：

```bash
firewall-cmd --get-services
firewall-cmd --permanent --zone=public --add-service=http
firewall-cmd --permanent --zone=public --add-service=https
firewall-cmd --reload
```

服务方式比端口方式更语义化，适合常见服务；非标准端口仍然需要 `--add-port` 或富规则。

## 后台运行程序

在命令末尾加 `&`，可以把程序放到后台运行：

```bash
ping www.qq.com &
```

如果不希望输出继续刷屏，可以重定向：

```bash
ping www.qq.com >>11.txt 2>&1 &
```

查看当前 Shell 的后台任务：

```bash
jobs -l
```

恢复到前台：

```bash
fg %1
```

结束后台任务：

```bash
kill PID
```

`&` 只是让命令进入后台，不保证用户退出 Shell 后继续运行。

## nohup

`nohup` 的作用是让进程忽略 HUP 信号。退出终端时，系统会向关联进程发送 HUP 信号，进程收到后通常会退出。

常用写法：

```bash
nohup ping www.qq.com >>11.txt 2>&1 &
```

关系拆分：

| 部分              | 作用             |
| --------------- | -------------- |
| `nohup`         | 忽略 HUP 信号      |
| `>>11.txt 2>&1` | 把标准输出和错误输出写入文件 |
| `&`             | 放入后台运行         |

查看进程：

```bash
ps -ef | grep ping
```

更完整的后台持久运行方案可以使用：

```bash
nohup command > app.log 2>&1 &
disown
```

长期服务更适合写成 `systemd` service，而不是长期依赖 `nohup`。

## 网络服务与端口

系统服务主要给本机使用，网络服务面向其他客户端。网络服务暴露给外部时，需要协议和端口。

系统端口映射文件：

```bash
less /etc/services
```

常见端口：

| 服务     | 端口        |
| ------ | --------- |
| FTP 数据 | `20/tcp`  |
| FTP 控制 | `21/tcp`  |
| SSH    | `22/tcp`  |
| Telnet | `23/tcp`  |
| HTTP   | `80/tcp`  |
| HTTPS  | `443/tcp` |

实际排查端口时，以服务当前监听为准：

```bash
ss -lntp
lsof -i:22
```

## 系统信息查看

### uptime

```bash
uptime
uptime -p
uptime -s
```

输出中的 `load average` 是 1 分钟、5 分钟、15 分钟平均负载。负载需要结合 CPU 核心数判断，不能只看数值大小。

### free

```bash
free -h
free -s 5
```

字段：

| 字段           | 含义        |
| ------------ | --------- |
| `total`      | 总内存       |
| `used`       | 已使用       |
| `free`       | 空闲        |
| `shared`     | 共享内存      |
| `buff/cache` | 缓冲和缓存     |
| `available`  | 系统估算的可用内存 |

`available` 通常比 `free` 更适合判断系统还能不能分配内存。

### lscpu

```bash
lscpu
lscpu -e
```

`lscpu -e` 可以显示 CPU 编号、核心、Socket 等信息。

### last

```bash
last
```

字段示例：

| 字段      | 含义       |
| ------- | -------- |
| 用户名     | 登录用户     |
| `pts/0` | 虚拟终端     |
| IP 地址   | 登录来源     |
| 时间段     | 登录和退出时间  |
| 括号      | 本次登录持续时间 |

## 网络与磁盘监控

### iftop

`iftop` 用来查看实时网络带宽。

安装：

```bash
yum install epel-release -y
yum install iftop -y
```

常用命令：

```bash
iftop -Bnt -i ens33
```

| 参数   | 含义           |
| ---- | ------------ |
| `-i` | 指定网卡         |
| `-n` | 不解析域名，只看 IP  |
| `-t` | 文本输出         |
| `-B` | 以 Byte 为单位显示 |

类似工具还有 `nethogs`，适合按进程查看网络流量。

### iostat

`iostat` 来自 `sysstat` 包，用来观察 CPU 和磁盘 I/O。

```bash
yum install sysstat -y
iostat
iostat 1
iostat -x 1
```

常见字段：

| 字段          | 含义              |
| ----------- | --------------- |
| `%user`     | 用户态 CPU         |
| `%system`   | 内核态 CPU         |
| `%iowait`   | 等待 I/O 的 CPU 时间 |
| `%steal`    | 虚拟化环境中被宿主机占用的时间 |
| `%idle`     | 空闲 CPU          |
| `tps`       | 每秒 I/O 请求次数     |
| `kB_read/s` | 每秒读取量           |
| `kB_wrtn/s` | 每秒写入量           |
| `kB_read`   | 总读取量            |
| `kB_wrtn`   | 总写入量            |

磁盘写入测试示例：

```bash
dd if=/dev/zero of=./a.dat bs=1k count=1M oflag=direct
dd if=/dev/zero of=./b.dat bs=1G count=1 oflag=direct
```

### iotop

`iotop` 用于实时查看哪个进程在产生磁盘 I/O。

```bash
yum install iotop -y
iotop
iotop -o
```

## SSH 密钥登录

生成密钥：

```bash
ssh-keygen
```

默认会在 `~/.ssh/` 下生成：

| 文件           | 作用           |
| ------------ | ------------ |
| `id_rsa`     | 私钥，必须保护好     |
| `id_rsa.pub` | 公钥，可以分发到远程主机 |

发送公钥：

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub root@远程主机IP
```

验证连接：

```bash
ssh root@远程主机IP
```

SSH 配置文件：

```bash
vim /etc/ssh/sshd_config
```

常见配置：

```conf
PubkeyAuthentication yes
PasswordAuthentication no
GSSAPIAuthentication no
UseDNS no
```

修改后先检查配置：

```bash
sshd -t
```

再重启 SSH：

```bash
systemctl restart sshd
```

注意：正式禁用密码登录前，必须确认密钥登录已经可以正常使用，并保留当前 SSH 会话，避免把自己锁在服务器外面。

## rsync 远程同步

`rsync` 使用增量复制，适合远程同步文件。通过 SSH 使用时，可以分为推送和拉取。

### 常用参数

| 参数               | 含义                    |
| ---------------- | --------------------- |
| `-a`             | 归档模式，递归并保持权限、属主、时间等属性 |
| `-v`             | 显示详细过程                |
| `-z`             | 传输时压缩                 |
| `-P`             | 显示进度并保留部分传输文件         |
| `--partial`      | 保留未传完的大文件，便于断点续传      |
| `--bwlimit`      | 限速                    |
| `--exclude`      | 排除文件或目录               |
| `--exclude-from` | 从文件读取排除规则             |
| `--delete`       | 删除目标端多余文件，使两端一致       |
| `--dry-run`      | 预演，不实际修改              |

### 拉取

```bash
rsync -av -e 'ssh -p 22' root@远程主机IP:/etc/hosts /tmp
rsync -av root@远程主机IP:/etc/hosts /tmp
```

### 推送

```bash
rsync -av /tmp/1.txt root@远程主机IP:/tmp
```

### 目录末尾斜杠

| 写法                      | 含义                |
| ----------------------- | ----------------- |
| `rsync -av /new 目标目录/`  | 同步 `new` 目录本身     |
| `rsync -av /new/ 目标目录/` | 只同步 `new` 目录里面的内容 |

### 排除文件

```bash
rsync -avzP kkkk@192.168.0.177:backup/etc /tmp --exclude=rsync.password
rsync -avzP kkkk@192.168.0.177:backup/etc /tmp --exclude=wok
```

多个排除项可以写入文件：

```bash
rsync -avzP --exclude-from=exclude.txt 源 目标
```

大量小文件传输时，rsync 可能效率较低，甚至卡住。可以先用 `tar` 打包，再传输压缩包。

## 易错点

| 问题                      | 说明                            |
| ----------------------- | ----------------------------- |
| `--permanent` 后规则没立即生效  | 还需要执行 `firewall-cmd --reload` |
| `reload` 后临时规则消失        | 未保存为永久规则                      |
| source 和 interface 同时存在 | source 匹配优先                   |
| 一个 source 绑定多个 zone     | 不允许，需先移除旧绑定                   |
| `nohup` 不等于后台运行         | 后台运行由 `&` 完成                  |
| 禁用 SSH 密码登录前没验证密钥       | 容易造成无法远程登录                    |
| rsync 目录末尾 `/`          | 会影响是否同步目录本身                   |
| `--delete`              | 会删除目标端多余文件，使用前建议先 `--dry-run` |
