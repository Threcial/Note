---
title: CentOS 7 安装初始化与文本处理工具
type: knowledge
status: evergreen
source: day10.md
created: 2026-06-13
updated: 2026-06-13
tags:
  - Linux
  - CentOS7
  - 系统安装
  - 系统初始化
  - VMware
  - sort
  - uniq
  - wc
  - grep
  - sed
  - awk
  - 正则表达式
aliases:
  - DAY10
  - CentOS7 初始化
  - Linux 文本处理
  - 文本三剑客
Belongs to:
  - "[[Linux 运维基础]]"
  - "[[CentOS 7]]"
  - "[[文本处理]]"
Related to:
  - "[[DAY01 - CentOS 7 基础环境与常用命令]]"
  - "[[DAY06 - RPM、YUM、源码安装与网络基础]]"
  - "[[DAY09 - firewalld、防火墙区域、后台任务与远程传输]]"
  - "[[YUM 软件包管理]]"
  - "[[SSH 密钥登录]]"
  - "[[VMware 克隆]]"
Has:
  - "[[U盘安装 CentOS 7]]"
  - "[[CentOS 7 初始化]]"
  - "[[大于 2T 磁盘安装]]"
  - "[[sort]]"
  - "[[uniq]]"
  - "[[wc]]"
  - "[[grep]]"
  - "[[sed]]"
  - "[[awk]]"
  - "[[正则表达式]]"
_organized: true
---
# CentOS 7 安装初始化与文本处理工具

## 概览

| 主题       | 结论                                                       |
| -------- | -------------------------------------------------------- |
| U 盘安装    | 启动参数中的 LABEL 必须与 U 盘卷标匹配                                 |
| 大于 2T 磁盘 | MBR 无法完整管理大于 2T 的磁盘，应使用 GPT，并根据启动方式配置 EFI 或 BIOS Boot 分区 |
| 初始化      | 最小化安装后需要处理 yum 源、基础工具、时间同步、文件描述符、SSH、安全策略                |
| 文本处理     | `sort/uniq/wc` 负责排序、去重、统计；`grep/sed/awk` 负责文本查找、编辑和取列    |

## U 盘安装 CentOS 7

安装时需要先确认 U 盘卷标。进入安装界面后按 `Tab` 修改启动参数，将其中类似下面的内容：

```text
LABEL=Centos\x207\x20x\86_64 quiet
```

改成与 U 盘实际卷标一致的值，否则安装程序可能找不到安装介质。

安装选择建议：

| 选项   | 建议                                |
| ---- | --------------------------------- |
| 语言   | 使用 English，避免中文环境造成日志、路径或终端显示问题   |
| 安装类型 | 最小化安装                             |
| 软件选择 | 除 `smart card support` 外，根据实际需要选择 |
| 网络   | 安装时打开网络，便于后续初始化                   |

## 大于 2T 磁盘安装

MBR 分区表最大只能识别约 2TiB 空间。大于 2T 的磁盘建议使用 GPT 分区表。

| 启动方式       | 必要分区                                                |
| ---------- | --------------------------------------------------- |
| UEFI + GPT | EFI System Partition，挂载到 `/boot/efi`，常见大小 500M      |
| BIOS + GPT | BIOS Boot Partition，常见大小 1M 到 2M，部分环境会给到 500M 以简化处理 |
| BIOS + MBR | 不适合完整使用大于 2T 磁盘                                     |

如果分区方式与启动方式不匹配，可能安装完成后找不到启动盘，系统无法启动。

## 网卡检查

服务器多网卡时，先确认网线插在哪个网口：

```bash
ip addr
ip link
```

如果网卡状态中出现：

```text
NO-CARRIER
```

通常表示该网口没有插网线、网线损坏，或者对端交换机端口没有正常工作。

网络不通的基础检查顺序：

1. 网线和交换机端口。
2. 网卡是否 UP。
3. IP、网关、DNS 是否正确。
4. 路由是否正确。
5. 防火墙和安全组是否拦截。

## CentOS 7 初始化

### 关闭 SELinux

配置文件：

```bash
vim /etc/selinux/config
```

修改：

```ini
SELINUX=disabled
```

临时关闭：

```bash
setenforce 0
```

是否关闭 SELinux 取决于环境。如果是实验环境或已有其他安全策略，可以关闭；生产环境更推荐理解策略后再决定。

### 防火墙策略

如果是云主机，云安全组已经做了访问控制，系统防火墙可根据实际情况关闭：

```bash
systemctl disable firewalld.service
systemctl stop firewalld.service
```

如果本机需要承担边界控制，优先使用 `firewalld` 规则精确开放端口，不建议直接关闭。

### 更换 yum 仓库

阿里云 CentOS 7 源：

```bash
curl -s -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
```

如果不可用，可使用其他镜像源，例如科大源：

```bash
curl -o /etc/yum.repos.d/Centos7-ustc.repo https://mirrors.wlnmp.com/centos/Centos7-ustc-x86_64.repo
```

如果新源与旧源冲突，清理 `/etc/yum.repos.d/` 中冲突的 `.repo` 文件。

安装 EPEL：

```bash
yum install epel-release -y
```

EPEL 使用阿里云镜像：

```bash
curl -s -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```

重新生成缓存：

```bash
yum clean all
yum makecache
```

### 安装基础软件

```bash
yum groupinstall "Compatibility libraries" "Base" "Development tools" -y
yum groupinstall "debugging Tools" "Dial-up Networking Support" -y
yum install tree nmap dos2unix lrzsz nc lsof wget tcpdump htop iftop iotop sysstat nethogs -y
yum install psmisc net-tools bash-completion vim-enhanced -y
yum update -y
```

常见用途：

| 工具          | 用途                           |
| ----------- | ---------------------------- |
| `tree`      | 树形查看目录                       |
| `nmap`      | 端口扫描                         |
| `dos2unix`  | 转换 Windows 换行符               |
| `lrzsz`     | 终端上传下载                       |
| `nc`        | 端口探测                         |
| `lsof`      | 查看文件与端口占用                    |
| `tcpdump`   | 抓包                           |
| `htop`      | 交互式进程查看                      |
| `iftop`     | 网络流量查看                       |
| `iotop`     | 磁盘 I/O 进程查看                  |
| `sysstat`   | `iostat`、`sar` 等系统性能工具       |
| `nethogs`   | 按进程查看网络流量                    |
| `net-tools` | 提供 `ifconfig`、`netstat` 等旧命令 |

### 精简开机服务

原始命令：

```bash
systemctl list-unit-files |grep enable|egrep -v "sshd.service|crond.service|sysstat|rsyslog|^NetworkManager.service|irqbalance.servide"|awk '{print "systemctl disable",$1}'|bash
```

该命令会批量禁用不在白名单中的开机启动服务。使用前建议先输出确认，不要直接管道给 `bash`：

```bash
systemctl list-unit-files | grep enabled | egrep -v "sshd.service|crond.service|sysstat|rsyslog|^NetworkManager.service|irqbalance.service"
```

确认后再执行禁用。

保留的基础服务通常包括：

```text
autovt@.service
crond.service
getty@.service
NetworkManager.service
rsyslog.service
sshd.service
sysstat.service
```

### 时间同步

原始方式使用 `ntpdate`：

```cron
0 */12 * * * /usr/sbin/ntpdate ntp3.aliyun.com >/dev/null 2>&1
```

CentOS 7 更常见的长期方案是使用 `chronyd`：

```bash
systemctl enable chronyd --now
chronyc sources -v
timedatectl status
```

如果仍使用 `ntpdate`，适合简单实验环境；长期运行的服务器更推荐 chrony。

### 文件描述符限制

查看当前限制：

```bash
ulimit -n
```

修改：

```bash
vim /etc/security/limits.conf
```

添加：

```conf
*                -       nofile          65535
```

重新登录后验证：

```bash
ulimit -n
```

高并发服务需要同时关注 systemd service 中的 `LimitNOFILE`，否则服务进程未必继承登录 Shell 的限制。

### 内核参数优化

编辑：

```bash
vim /etc/sysctl.conf
```

示例：

```conf
net.ipv4.tcp_keepalive_time = 1800
net.ipv4.ip_local_port_range = 4000 65000
net.ipv4.tcp_max_syn_backlog = 8192
net.core.somaxconn = 16384
net.core.netdev_max_backlog = 16384
```

立即加载：

```bash
sysctl -p
```

查看是否生效：

```bash
cat /proc/sys/net/ipv4/tcp_keepalive_time
cat /proc/sys/net/ipv4/ip_local_port_range
cat /proc/sys/net/ipv4/tcp_max_syn_backlog
cat /proc/sys/net/core/somaxconn
cat /proc/sys/net/core/netdev_max_backlog
```

这些参数用于增强网络连接承载能力，但不应脱离业务负载盲目套用。

### 清除登录显示信息

```bash
>/etc/issue
>/etc/issue.net
```

验证：

```bash
cat /etc/issue
cat /etc/issue.net
```

### SSH 初始化

生成密钥：

```bash
ssh-keygen
```

修改 SSH 配置：

```bash
vim /etc/ssh/sshd_config
```

建议先保留密码登录，等所有管理端都验证密钥登录成功后，再禁用密码登录。

```conf
PubkeyAuthentication yes
PasswordAuthentication yes
GSSAPIAuthentication no
UseDNS no
```

验证配置：

```bash
sshd -t
systemctl restart sshd
```

确认密钥登录正常后，再修改：

```conf
PasswordAuthentication no
```

### 锁定关键系统文件

```bash
chattr +i /etc/passwd
chattr +i /etc/shadow
```

查看：

```bash
lsattr /etc/passwd /etc/shadow
```

解除：

```bash
chattr -i /etc/passwd
chattr -i /etc/shadow
```

注意：锁定这些文件会影响用户创建、密码修改、软件安装脚本等操作。生产环境使用前必须确认维护流程。

## VMware 克隆机处理

克隆完成后需要处理 MAC、主机名、IP 和 UUID。

### 修改 MAC

启动虚拟机前，在 VMware 网卡高级设置中重新生成 MAC 地址。启动后验证：

```bash
ip addr
```

### 修改主机名

```bash
vim /etc/hostname
hostnamectl set-hostname 新主机名
```

### 修改网卡配置

```bash
vim /etc/sysconfig/network-scripts/ifcfg-ens33
```

需要修改：

```ini
IPADDR=192.168.0.167
UUID="9949456b-9b9f-4010-aa47-283931b669c4"
```

生成新的 UUID：

```bash
uuidgen
```

UUID 不建议手写，应使用 `uuidgen` 生成。

重启网络或系统后验证：

```bash
ip addr
ping 网关
```

## sort

`sort` 用于按行排序。

```bash
sort [选项] 文件
```

| 参数   | 含义         |
| ---- | ---------- |
| `-c` | 检查文件是否已经排序 |
| `-t` | 指定分隔符      |
| `-k` | 指定排序字段     |
| `-n` | 按数值排序      |
| `-r` | 反向排序       |
| `-o` | 输出到指定文件    |

示例：

```bash
sort passwd
sort -c passwd
sort -t ':' -k 3 passwd
sort -t ':' -k 3 -n passwd
sort -t ':' -k 3 -r -n passwd
sort -t ':' -k 4 passwd -o /root/passwd
```

不能用普通重定向把排序结果写回原文件：

```bash
sort passwd > passwd
```

这种写法可能先清空文件。写回原文件必须使用：

```bash
sort passwd -o passwd
```

## uniq

`uniq` 用于处理相邻重复行。实际使用时通常先排序：

```bash
sort 文件 | uniq
```

| 参数   | 含义         |
| ---- | ---------- |
| `-c` | 显示重复次数     |
| `-d` | 只显示重复行     |
| `-u` | 只显示只出现一次的行 |

示例：

```bash
sort passwd | uniq -c
sort passwd | uniq -d
sort passwd | uniq -u
```

`uniq` 只能识别相邻重复行，不排序时可能漏统计。

## wc

`wc` 用于统计行数、单词数、字符数。

```bash
wc [选项] 文件
```

| 参数   | 含义          |
| ---- | ----------- |
| `-l` | 行数，空行也统计    |
| `-w` | 单词数，以空白字符分隔 |
| `-m` | 字符数         |

示例：

```bash
wc hello.sh
wc -l hello.sh
wc -w hello.sh
wc -m hello.sh
```

## grep

`grep` 用于按行查找匹配内容。

```bash
grep [选项] '模式' 文件
```

| 参数             | 含义        |
| -------------- | --------- |
| `-v`           | 取反        |
| `-i`           | 忽略大小写     |
| `-n`           | 显示行号      |
| `-w`           | 按单词匹配     |
| `-o`           | 只显示匹配到的内容 |
| `-E`           | 使用扩展正则表达式 |
| `-r`           | 递归查找目录    |
| `--color=auto` | 高亮匹配内容    |

示例：

```bash
grep 'teacher' passwd
grep -v 'linux' passwd
grep -i 'LINUX' passwd
grep -n 'linux' passwd
grep -w 'linux' passwd
grep -o 'linux' passwd
grep -E 'linux|weibo' passwd
```

`egrep` 等价于 `grep -E`，实际使用中推荐直接写 `grep -E`。

## 正则表达式

正则表达式用于匹配文本模式，通常配合 `grep`、`sed`、`awk` 使用。普通 Shell 通配符和正则不是一回事。

### 基础正则 BRE

| 表达式      | 含义             |
| -------- | -------------- |
| `^`      | 行首             |
| `$`      | 行尾             |
| `^$`     | 空行             |
| `.`      | 任意一个字符，不匹配空行   |
| `\`      | 转义             |
| `*`      | 前一个字符重复 0 次或多次 |
| `.*`     | 任意长度内容         |
| `[abc]`  | 匹配集合中任意一个字符    |
| `[a-z]`  | 匹配连续范围         |
| `[^abc]` | 取反，不匹配集合内字符    |

示例：

```bash
grep '^l' linux.txt
grep '\!$' linux.txt
grep -n '^$' linux.txt
grep '\.$' linux.txt
grep 'll*' linux.txt
grep '^a.*a$' wei.txt
grep '[0-9a-z]' linux.txt
grep '[^a-c]' linux.txt
```

### 扩展正则 ERE

需要使用 `grep -E`、`sed -r` 或 `awk` 的正则语法。

| 表达式       | 含义                 |
| --------- | ------------------ |
| `+`       | 前一个字符出现 1 次或多次     |
| `?`       | 前一个字符出现 0 次或 1 次   |
| `         | `                  |
| `{n,m}`   | 前一个字符最少 n 次、最多 m 次 |
| `{n,}`    | 前一个字符至少 n 次        |
| `{n}`     | 前一个字符正好 n 次        |
| `{,m}`    | 前一个字符最多 m 次        |
| `()`      | 分组                 |
| `\1`、`\2` | 引用前面的分组            |

示例：

```bash
grep -E 'q+' linux.txt
grep -E 'go?d' linux.txt
grep -E 'qq|linux' linux.txt
grep -E 'l{1,2}' linux.txt
grep -E 'g(oo|ww)d' linux.txt
grep -E '(good)(gwwd)\2' linux.txt
```

## sed

`sed` 是流编辑器，适合按行对文本做查找、删除、替换、插入和追加。

```bash
sed [选项] '动作' 文件
```

| 参数   | 含义               |
| ---- | ---------------- |
| `-n` | 取消默认输出，常与 `p` 配合 |
| `-i` | 直接修改文件           |
| `-e` | 多次编辑             |
| `-r` | 使用扩展正则表达式        |

| 内置动作 | 含义      |
| ---- | ------- |
| `p`  | 打印      |
| `d`  | 删除      |
| `s`  | 替换      |
| `g`  | 全局替换    |
| `a`  | 在指定行后追加 |
| `i`  | 在指定行前插入 |

示例：

```bash
sed -n '2,3p' linux.txt
sed -n '1p;3p' linux.txt
sed -n '/linux/p' linux.txt
sed '/linux/d' linux.txt
sed 's#linux#windows#g' linux.txt
sed -i 's#linux#windows#g' linux.txt
sed -e 's#linux#windows#g' -e 's#llll#aaaa#g' linux.txt
sed '2a hello' linux.txt
sed '2i hello' linux.txt
sed '3,5d' linux.txt
sed '2d;4d' wei.txt
```

修改文件前建议备份：

```bash
sed -i.bak 's#old#new#g' file
```

### sed 实战：取网卡 IP

```bash
ip a | sed -n '9p' | sed 's#^.*inet ##g' | sed 's#/24.*$##g'
ip a | sed -n '9p' | sed -e 's#^.*inet ##g' -e 's#/24.*$##g'
ip a | sed -rn '9s#^.*inet (.*)/24.*$#\1#gp'
```

更稳的写法应避免依赖固定行号，例如：

```bash
ip -o -4 addr show dev ens33 | awk '{print $4}' | cut -d/ -f1
```

### sed 实战：取文件权限

```bash
stat /etc/hosts | sed -n '4p' | sed 's#^.*(0##g' | sed 's#/-.*$##g'
stat /etc/hosts | sed -rn '4s#^.*\(0(.*)\/-.*$#\1#gp'
```

更直接的写法：

```bash
stat -c '%a' /etc/hosts
```

## awk

`awk` 适合处理列，也可以作为一门小型文本处理语言使用。

```bash
awk [选项] '条件 {动作}' 文件
```

| 参数或变量     | 含义     |
| --------- | ------ |
| `-F`      | 指定分隔符  |
| `$1`      | 第一列    |
| `$0`      | 整行     |
| `$NF`     | 最后一列   |
| `$(NF-1)` | 倒数第二列  |
| `NR`      | 当前行号   |
| `~`       | 某列匹配正则 |
| `print`   | 输出     |

示例：

```bash
awk -F ":" '{print $3}' test.txt
awk -F ":" '{print $3,$5}' test.txt
awk -F ":" '{print $NF}' test.txt
awk -F ":" '{print $(NF-1)}' test.txt
awk 'NR==3' passwd
awk 'NR==3;NR==5' passwd
awk 'NR>2&&NR<7' passwd
awk 'NR==3,NR==6' passwd
awk -F ":" 'NR==2{print $3}' passwd
awk '/lp/' test.txt
awk -F ":" '/shutdown/{print $NF}' passwd
awk '/^[^s]/' test.txt
awk -F ":" '$1~/sync/ {print $NF}' test.txt
```

### awk 实战：取网卡 IP

```bash
ip a | awk 'NR==9{print $2}' | awk -F "/" '{print $1}'
```

更稳写法：

```bash
ip -o -4 addr show dev ens33 | awk '{print $4}' | awk -F/ '{print $1}'
```

### awk 实战：打印分数大于 70 的行

```bash
awk '$NF>70 {print $0}' a.txt
```

## grep、sed、awk 的分工

| 工具     | 适合做什么           |
| ------ | --------------- |
| `grep` | 找行、过滤行          |
| `sed`  | 按行编辑、替换、删除、插入   |
| `awk`  | 按列处理、格式化输出、简单计算 |

典型组合：

```bash
grep 'error' app.log | awk '{print $1,$2,$NF}'
ip -o -4 addr show dev ens33 | awk '{print $4}' | cut -d/ -f1
stat -c '%a %U %G %n' /etc/hosts
```

## 易错点

| 问题             | 说明                                   |
| -------------- | ------------------------------------ |
| `uniq` 不先排序    | 只能处理相邻重复行                            |
| `sort > 原文件`   | 可能清空原文件，写回原文件用 `sort -o`             |
| Shell 通配符和正则混用 | `*` 在通配符和正则中的含义不同                    |
| `sed -i` 直接改文件 | 建议先不加 `-i` 验证，或使用 `-i.bak`           |
| 取网卡 IP 依赖固定行号  | 网卡、系统、地址数量不同会导致行号变化                  |
| `awk` 默认分隔符    | 默认按空白分割，处理 `/etc/passwd` 要用 `-F ":"` |
