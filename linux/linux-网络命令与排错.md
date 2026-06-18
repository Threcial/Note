---
type: Reference
status: Organized
Belongs to: "[[Network]]"
Related to:
  - "[[linux-网络基础]]"
Has:
  - "[[ss]]"
  - "[[curl]]"
  - "[[ip 命令]]"
  - "[[ping]]"
  - "[[nc]]"
  - "[[traceroute]]"
  - "[[TCP 三次握手]]"
source: ""
created: 2026-06-13
updated: 2026-06-13
---

# 网络命令与排错

## 网络排错链路

| 层次 | 关注点 | 常用命令 |
| --- | --- | --- |
| 网卡 | 网卡是否 UP、是否有地址 | `ip addr`、`ip link` |
| 路由 | 默认网关、目标网段路径 | `ip route`、`route -n` |
| 连通性 | ICMP 是否通、丢包 | `ping`、`traceroute` |
| DNS | 域名是否能解析 | `nslookup`、`dig` |
| 端口 | 服务端口是否监听、远端端口是否可达 | `ss`、`nc`、`telnet` |
| 进程 | 哪个进程占用端口 | `ss -lntp`、`lsof -i:端口` |
| 服务 | 服务是否启动 | `systemctl status` |

## 旧命令与新命令

| 旧命令 | 更推荐 | 说明 |
| --- | --- | --- |
| `ifconfig` | `ip addr` | 需要 `net-tools` |
| `route -n` | `ip route show` | 更完整 |
| `netstat -lntp` | `ss -lntp` | 大量连接时更快 |
| `telnet ip port` | `nc -vz ip port` | 仅测连通性更直接 |

## TCP 连接状态

### 三次握手

| 步骤 | 客户端 | 服务端 |
| --- | --- | --- |
| 第一次 | `SYN_SENT` → 发送 SYN | `LISTEN` |
| 第二次 | `SYN_SENT` | `SYN_RECV` → 回复 SYN+ACK |
| 第三次 | `ESTABLISHED` → 回复 ACK | `ESTABLISHED` |

### 四次挥手

| 步骤 | 主动关闭方 | 被动关闭方 |
| --- | --- | --- |
| 第一次 | → FIN | 接收 FIN |
| 第二次 | ← ACK | → 回复 ACK |
| 第三次 | ← FIN | → 发送 FIN |
| 第四次 | → ACK | 接收 ACK |

`TIME_WAIT` 出现在主动关闭方，作用是保证最后一个 ACK 有机会被对端收到。

### TCP 状态排查

| 状态 | 排查意义 |
| --- | --- |
| `LISTEN` | 服务是否对外提供入口 |
| `SYN_SENT` | 常见于远端不可达、防火墙拦截 |
| `ESTABLISHED` | 正常通信 |
| `TIME_WAIT` | 短连接服务常见，需结合业务判断 |
| `CLOSE_WAIT` | 大量出现通常是应用没有正确关闭连接 |

## netstat 与 ss

### netstat

```bash
netstat -lntp     # TCP 监听端口
netstat -antp     # 所有 TCP 连接
```

### ss（推荐）

```bash
ss -lntp          # TCP 监听端口和对应进程
ss -antp          # 所有 TCP 连接
ss -lunp          # UDP 监听端口
ss -s             # 连接统计概要
```

## 网络配置

```bash
ip addr
ip a
ip addr show dev ens33
ip link show
ip neigh           # ARP 邻居表
```

临时添加 IP：

```bash
ip addr add 192.168.8.5/24 dev ens33
ip addr del 192.168.8.5/24 dev ens33
```

### 修改网卡配置

```ini
BOOTPROTO=static
IPADDR=192.168.0.168
NETMASK=255.255.255.0
GATEWAY=192.168.0.1
DNS1=192.168.0.1
ONBOOT=yes
```

```bash
ifdown ens33 && ifup ens33
```

## wget 与 curl

### wget

```bash
wget URL
wget -O 文件名 URL
wget -c --limit-rate=1m URL   # 断点续传、限速
```

### curl

```bash
curl -I https://example.com           # 查看响应头
curl -o /dev/null -w "%{http_code}" URL
curl -s -o /dev/null -w "%{http_code} %{time_total}\n" URL
```

## 探测命令

```bash
ping -c 4 192.168.1.1
telnet 192.168.1.10 80
nc -vz 192.168.1.10 80
nslookup www.baidu.com
dig www.baidu.com
traceroute 8.8.8.8
```

## 易错点

| 问题 | 说明 |
| --- | --- |
| `netstat` 没有命令 | 安装 `net-tools`，但更推荐 `ss` |
| `ifconfig` 没有命令 | 推荐使用 `ip addr` |
| `TIME_WAIT` 很多不一定是故障 | 短连接服务常见，需结合业务判断 |
| `CLOSE_WAIT` 很多更值得关注 | 通常说明应用没有正确关闭连接 |
