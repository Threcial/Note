---
type: Reference
status: Organized
Belongs to: "[[Network]]"
Has:
  - "[[firewalld]]"
  - "[[firewall-cmd]]"
source: ""
created: 2026-06-13
updated: 2026-06-13
---

# firewalld 防火墙

CentOS 7 的 firewalld 引入区域概念，不同区域代表不同信任级别。

## 区域

| 区域 | 默认策略与用途 |
| --- | --- |
| `drop` | 丢弃所有进入本机的包，不回复 |
| `block` | 拒绝进入，并回复拒绝信息 |
| `public` | 默认区域，通常只允许 SSH 和 dhcpv6-client |
| `external` | 面向外部网络，NAT 或网关场景 |
| `dmz` | 隔离区，只开放少量入口 |
| `trusted` | 接受所有连接 |

`drop` 和 `block` 的区别：`drop` 不回复，`block` 回复拒绝信息。

## firewall-cmd 常用参数

```bash
firewall-cmd --get-default-zone
firewall-cmd --set-default-zone=<zone>
firewall-cmd --get-zones
firewall-cmd --list-all
firewall-cmd --list-all-zones
firewall-cmd --get-active-zones
firewall-cmd --add-source=<IP/CIDR>
firewall-cmd --add-interface=<网卡>
firewall-cmd --change-interface=<网卡>
firewall-cmd --add-port=80/tcp
firewall-cmd --remove-port=80/tcp
firewall-cmd --add-protocol=icmp
firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.0.81" port protocol="tcp" port="22" accept'
firewall-cmd --reload
firewall-cmd --panic-on
```

## 运行时规则与永久规则

```bash
# 临时生效（重启丢失）
firewall-cmd --zone=drop --add-port=80/tcp

# 永久生效
firewall-cmd --permanent --zone=drop --add-port=80/tcp
firewall-cmd --reload

# 当前运行时保存为永久
firewall-cmd --runtime-to-permanent
```

## zone 匹配逻辑

| 优先级 | 匹配依据 | 说明 |
| --- | --- | --- |
| 1 | source | 来源地址绑定的 zone 优先 |
| 2 | interface | 流量进入的网卡所属 zone |
| 3 | default zone | 默认区域 |

一个 source 只能绑定到一个 zone。

## 常用配置示例

### 网卡切到 drop 区域

```bash
firewall-cmd --permanent --zone=drop --change-interface=ens33
firewall-cmd --reload
```

### 只允许指定 IP 访问 22 端口

```bash
firewall-cmd --zone=drop --permanent --add-rich-rule='rule family="ipv4" source address="192.168.0.81" port protocol="tcp" port="22" accept'
firewall-cmd --reload
```

### 使用 service 方式开放服务

```bash
firewall-cmd --get-services
firewall-cmd --permanent --zone=public --add-service=http
firewall-cmd --reload
```

## 易错点

| 问题 | 说明 |
| --- | --- |
| `--permanent` 后规则没立即生效 | 还需 `firewall-cmd --reload` |
| `reload` 后临时规则消失 | 未保存为永久规则 |
| source 和 interface 同时存在 | source 匹配优先 |
| 一个 source 绑定多个 zone | 不允许，需先移除旧绑定 |
