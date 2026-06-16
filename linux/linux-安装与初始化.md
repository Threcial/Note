---
type: knowledge
status: evergreen
created: 2026-06-13
updated: 2026-06-13
related_to:
  - "[[Linux 运维基础]]"
  - "[[CentOS 7]]"
  - "[[linux-软件包管理]]"
_organized: true
---

# 安装与初始化

## U 盘安装 CentOS 7

安装时需要确认 U 盘卷标。进入安装界面后按 `Tab` 修改启动参数，将 `LABEL=Centos\x207\x20x\86_64` 改成与 U 盘实际卷标一致。

安装选择建议：

| 选项 | 建议 |
| --- | --- |
| 语言 | English，避免中文环境问题 |
| 安装类型 | 最小化安装 |
| 网络 | 安装时打开网络 |

## 大于 2T 磁盘安装

MBR 分区表最大识别约 2TiB，大于 2T 的磁盘应使用 GPT。

| 启动方式 | 必要分区 |
| --- | --- |
| UEFI + GPT | EFI System Partition，挂载 `/boot/efi` |
| BIOS + GPT | BIOS Boot Partition |

## 网卡检查

```bash
ip addr
ip link
```

`NO-CARRIER` 表示该网口没有插网线或对端端口异常。

## 系统初始化

### 关闭 SELinux

```bash
vim /etc/selinux/config
SELINUX=disabled
setenforce 0
```

### 防火墙策略

```bash
systemctl disable firewalld.service
systemctl stop firewalld.service
```

如果本机需要承担边界控制，优先使用 firewalld 规则精确开放端口。

### 更换 yum 仓库

```bash
curl -s -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum install epel-release -y
curl -s -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum clean all && yum makecache
```

### 安装基础软件

```bash
yum groupinstall "Compatibility libraries" "Base" "Development tools" -y
yum install tree nmap dos2unix lrzsz nc lsof wget tcpdump htop iftop iotop sysstat nethogs -y
yum install psmisc net-tools bash-completion vim-enhanced -y
```

### 精简开机服务

```bash
systemctl list-unit-files | grep enabled | egrep -v "sshd.service|crond.service|sysstat|rsyslog|^NetworkManager.service|irqbalance.service"
```

确认后再管道给 `systemctl disable`。

### 时间同步

```bash
systemctl enable chronyd --now
chronyc sources -v
timedatectl status
```

### 文件描述符限制

```bash
vim /etc/security/limits.conf
```

```conf
*   -   nofile   65535
```

高并发服务还需关注 systemd service 中的 `LimitNOFILE`。

### 内核参数优化

```bash
vim /etc/sysctl.conf
```

```conf
net.ipv4.tcp_keepalive_time = 1800
net.ipv4.ip_local_port_range = 4000 65000
net.core.somaxconn = 16384
```

```bash
sysctl -p
```

### SSH 初始化

```bash
ssh-keygen
vim /etc/ssh/sshd_config
```

```conf
PubkeyAuthentication yes
PasswordAuthentication yes
GSSAPIAuthentication no
UseDNS no
```

确认密钥登录正常后再禁用密码登录。

### 锁定关键系统文件

```bash
chattr +i /etc/passwd
chattr +i /etc/shadow
```

## VMware 克隆机处理

```bash
# 修改主机名
hostnamectl set-hostname 新主机名

# 修改网卡配置
vim /etc/sysconfig/network-scripts/ifcfg-ens33
# 修改 IPADDR，使用 uuidgen 生成新 UUID

# 重启网络
systemctl restart network
```

## 易错点

| 场景 | 说明 |
| --- | --- |
| 创建分区时把 `+2G` 填到起始扇区 | 起始扇区回车，结束扇区填容量 |
| 时间同步后没写硬件时钟 | `hwclock -w` 写入硬件时钟 |
| VMware 克隆后网络异常 | 检查 MAC、UUID、主机名 |
