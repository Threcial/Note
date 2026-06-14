---
title: Nginx 优化、负载均衡与 Keepalived 高可用
type: knowledge
status: evergreen
source: day18.md
created: 2026-06-14
updated: 2026-06-14
tags:
  - Linux
  - CentOS7
  - Nginx
  - 性能优化
  - 负载均衡
  - Keepalived
  - 高可用
  - VRRP
aliases:
  - DAY18
  - Nginx 优化
  - Nginx 负载均衡
  - Keepalived 高可用
  - Nginx HA
Belongs to:
  - "[[Nginx]]"
  - "[[高可用架构]]"
Related to:
  - "[[Nginx 配置文件]]"
  - "[[Nginx 反向代理]]"
  - "[[TCP 三次握手]]"
  - "[[Linux 进程]]"
  - "[[systemd 服务管理]]"
  - "[[云负载均衡]]"
Has:
  - "[[worker_processes]]"
  - "[[worker_connections]]"
  - "[[gzip]]"
  - "[[keepalive]]"
  - "[[upstream]]"
  - "[[ip_hash]]"
  - "[[weight]]"
  - "[[backup]]"
  - "[[down]]"
  - "[[Keepalived]]"
  - "[[VRRP]]"
  - "[[VIP]]"
  - "[[裂脑]]"
_organized: true
---
# Nginx 优化、负载均衡与 Keepalived 高可用

Nginx 优化不是把参数改大，而是让配置和机器资源、业务流量、后端能力匹配。基础优化集中在 worker 进程、连接数、长连接、压缩、日志级别、上传大小和静态资源缓存。负载均衡集中在 `upstream` 与 `proxy_pass`。高可用集中在前端分发器的单点故障处理，常见方案是 Keepalived + VIP，云环境则优先使用云厂商 SLB/CLB/ELB。

## 总览

| 条目                   | 关系                       |
| -------------------- | ------------------------ |
| Nginx 优化             | 让连接、CPU、内存、I/O、带宽配置更匹配业务 |
| `worker_processes`   | 控制 worker 进程数量           |
| `worker_connections` | 控制每个 worker 的连接上限        |
| `keepalive_timeout`  | 控制长连接保留时间                |
| `gzip`               | 压缩文本类响应，节省带宽             |
| `upstream`           | 定义后端服务器组                 |
| `proxy_pass`         | 把请求代理到 upstream 或具体后端    |
| Keepalived           | 通过 VRRP 管理 VIP 漂移        |
| VIP                  | 对外提供的虚拟 IP               |

## Nginx 性能优化

### worker 进程

```nginx
worker_processes auto;
```

`auto` 会根据 CPU 核心数自动设置 worker 进程数。一般比手工写固定数字更稳妥。

### worker 连接数

```nginx
events {
    worker_connections 4096;
}
```

理论连接上限接近：

```text
worker_processes * worker_connections
```

实际还会受到系统文件描述符、连接状态、反向代理连接、内核参数影响。

查看当前文件描述符限制：

```bash
ulimit -n
```

临时调整：

```bash
ulimit -n 65535
```

systemd 管理的 Nginx 还要在 service 或 override 中设置：

```ini
[Service]
LimitNOFILE=65535
```

然后执行：

```bash
systemctl daemon-reload
systemctl restart nginx
```

### 上传大小

```nginx
client_max_body_size 100m;
```

可放在 `http`、`server`、`location`。默认通常较小，上传文件时报 413 时优先检查这个参数。

### 错误日志级别

```nginx
error_log /var/log/nginx/error.log error;
```

常用级别：

| 级别      | 说明             |
| ------- | -------------- |
| `warn`  | 警告及以上          |
| `error` | 错误及以上          |
| `crit`  | 严重错误           |
| `info`  | 信息很多，不建议生产长期开启 |
| `debug` | 调试用，生产谨慎使用     |

日志级别过低会增加磁盘 I/O，特别是在高并发环境中。

### 长连接

```nginx
keepalive_timeout 15;
keepalive_requests 8192;
```

长连接减少 TCP 三次握手和四次挥手开销，但时间过长会占用连接资源。默认 65 秒在部分场景偏长，面向公网业务可根据访问模式下调。

### sendfile 与零拷贝

```nginx
sendfile on;
```

静态文件传输中，`sendfile` 可以减少用户态和内核态之间的数据拷贝。

### gzip 压缩

```nginx
gzip on;
gzip_proxied any;
gzip_min_length 10k;
gzip_comp_level 6;
gzip_types text/plain text/css application/javascript application/json application/xml text/javascript;
```

适合压缩：

| 类型                   | 是否适合  |
| -------------------- | ----- |
| HTML/CSS/JS/JSON/XML | 适合    |
| 图片/视频/音频             | 通常不适合 |
| 已压缩文件 zip/gz/rar     | 不适合   |

压缩等级越高，CPU 消耗越大。常见生产环境取 `4` 到 `6`。

### 隐藏版本号

```nginx
server_tokens off;
```

这不是实质防护，但可以减少暴露版本信息。

### 静态资源缓存

```nginx
location ~* \.(png|gif|jpg|jpeg|webp|mp3|mp4|wav|avi)$ {
    expires 1h;
    root /usr/share/nginx/html;
}
```

如果资源文件名带 hash，可以把缓存时间拉长。普通文件名不建议缓存太久，否则更新后客户端可能仍拿旧资源。

## 连接数与性能观测

查看连接状态：

```bash
ss -antp | grep nginx
ss -ant | awk '{print $1}' | sort | uniq -c
```

查看 Nginx 已建立连接数：

```bash
ss -antp | grep nginx | grep ESTAB | wc -l
```

旧命令也可以用：

```bash
netstat -antpl | grep nginx | grep ESTABLISHED | wc -l
```

现代环境优先用 `ss`，速度更快，尤其是连接数很大时。

## 负载均衡

Nginx 七层负载均衡基于 `ngx_http_upstream_module`。核心结构是：前端 `server` 接请求，`proxy_pass` 转发到 `upstream` 服务器组。

### 轮询

```nginx
upstream web {
    server 192.168.0.120;
    server 192.168.0.121;
    server 192.168.0.122;
}

server {
    listen 80;

    location / {
        proxy_pass http://web;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

轮询按照请求顺序分发，未配置权重时各后端权重相同。

### 权重

```nginx
upstream web {
    server 192.168.0.120 weight=10;
    server 192.168.0.121 weight=3;
    server 192.168.0.122;
}
```

权重越大，请求分配越多。没有写 `weight` 时默认是 1。

### ip_hash

```nginx
upstream web {
    ip_hash;
    server 192.168.0.120;
    server 192.168.0.121;
}
```

`ip_hash` 可以让同一客户端 IP 尽量落到同一后端，用于缓解 session 不一致问题。更稳定的方式是应用层共享 session、Redis session、JWT 或无状态认证。

### 后端状态

```nginx
upstream web {
    server 192.168.0.120 down;
    server 192.168.0.121 backup;
    server 192.168.0.122 weight=5;
}
```

| 参数             | 含义             |
| -------------- | -------------- |
| `down`         | 不参与请求处理        |
| `backup`       | 其他后端不可用或繁忙时才参与 |
| `weight`       | 权重             |
| `max_fails`    | 失败次数阈值         |
| `fail_timeout` | 失败统计窗口         |

补充示例：

```nginx
upstream web {
    server 192.168.0.120 max_fails=3 fail_timeout=10s;
    server 192.168.0.121 max_fails=3 fail_timeout=10s;
}
```

## 负载均衡分发方式

### 基于域名

```nginx
upstream web1 {
    server 192.168.0.120;
    server 192.168.0.121;
}

upstream web2 {
    server 192.168.0.122;
    server 192.168.0.123;
}

server {
    listen 80;
    server_name www.abc.com;

    location / {
        proxy_pass http://web1;
    }
}

server {
    listen 80;
    server_name www.def.com;

    location / {
        proxy_pass http://web2;
    }
}
```

### 基于端口

```nginx
server {
    listen 8000;

    location / {
        proxy_pass http://web1;
    }
}

server {
    listen 80;

    location / {
        proxy_pass http://web2;
    }
}
```

### 基于目录

```nginx
server {
    listen 80;
    server_name www.abc.com;

    location ~* "^/xyj.*$" {
        proxy_pass http://xyj;
    }

    location / {
        proxy_pass http://swk;
    }
}
```

### 基于文件类型

```nginx
server {
    listen 80;
    server_name www.abc.com;

    location ~* \.html$ {
        proxy_pass http://html;
    }

    location ~* \.jsp$ {
        proxy_pass http://jsp;
    }
}
```

### 基于浏览器

```nginx
server {
    listen 80;

    location / {
        if ($http_user_agent ~* elinks) {
            proxy_pass http://elinks;
        }

        if ($http_user_agent ~* chrome) {
            proxy_pass http://chrome;
        }
    }
}
```

这类写法能跑，但不推荐长期使用。更清晰的方式是用 `map`：

```nginx
map $http_user_agent $backend {
    default chrome;
    ~*elinks elinks;
    ~*chrome chrome;
}

server {
    listen 80;

    location / {
        proxy_pass http://$backend;
    }
}
```

## Keepalived 高可用

### 适用场景

后端业务服务器故障时，Nginx upstream 可以剔除或绕开后端；但前端 Nginx 分发器本身也可能成为单点。Keepalived 通过 VRRP 维护一个 VIP，让主分发器故障后，备分发器接管 VIP。

云主机通常不支持自己随意漂移内网/公网 VIP。云环境优先使用云厂商提供的 SLB、CLB、ELB、NLB 等产品。

### MASTER 配置

```nginx
global_defs {
    router_id wb01
}

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh"
    interval 1
    weight -50
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 33
    priority 150
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1111
    }

    track_script {
        chk_nginx
    }

    virtual_ipaddress {
        192.168.31.33/24 dev ens33 label ens33:3
    }
}
```

### BACKUP 配置

```nginx
global_defs {
    router_id wb02
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 33
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        192.168.31.33/24 dev ens33 label ens33:3
    }
}
```

关键点：

| 参数                  | 要求               |
| ------------------- | ---------------- |
| `router_id`         | 每台机器唯一           |
| `virtual_router_id` | 同一组必须一致          |
| `priority`          | MASTER 高于 BACKUP |
| `auth_pass`         | 同组一致             |
| `interface`         | VIP 所在网卡         |
| `virtual_ipaddress` | 两端一致             |

### Nginx 检测脚本

更直接的检测脚本：

```bash
#!/bin/bash

if ! pidof nginx >/dev/null 2>&1; then
    exit 1
fi

exit 0
```

保存为：

```bash
vim /etc/keepalived/nginx_check.sh
chmod +x /etc/keepalived/nginx_check.sh
```

启动：

```bash
systemctl enable --now keepalived
systemctl status keepalived
```

查看 VIP：

```bash
ip addr show dev ens33
```

### 高可用测试

```bash
systemctl stop keepalived
ip addr show dev ens33
```

主节点停止 Keepalived 后，VIP 应漂移到备节点。恢复后是否抢回 VIP，取决于优先级和是否配置了非抢占策略。

## 裂脑问题

裂脑是指两台 Keepalived 节点在一定时间内无法接收到对方 VRRP 心跳，导致双方都认为自己应该持有 VIP，最终同一个 VIP 被两台机器同时使用。

常见原因：

| 原因         | 说明                            |
| ---------- | ----------------------------- |
| 心跳网络异常     | VRRP 报文不通                     |
| 网卡或交换机异常   | 节点之间看不到彼此                     |
| 防火墙拦截 VRRP | VRRP 使用 IP 协议号 112            |
| 配置不一致      | `virtual_router_id`、认证、网卡等不一致 |

监控思路：在备节点上检测主节点服务可达性，如果主节点 80 端口可达，同时备节点持有 VIP，则触发报警。

```bash
ip addr show dev ens33 | grep '192.168.31.33' >/dev/null
has_vip=$?

timeout 2 bash -c '</dev/tcp/192.168.31.20/80' >/dev/null 2>&1
master_nginx=$?

if [ $has_vip -eq 0 ] && [ $master_nginx -eq 0 ]; then
    echo "split brain risk"
fi
```

## 命令速查

| 目的               | 命令                            |
| ---------------- | ----------------------------- |
| 查看 Nginx 连接      | `ss -antp                     |
| 查看连接状态统计         | `ss -ant                      |
| 查看目录大小           | `du -h --max-depth=1`         |
| 检查 Nginx 配置      | `nginx -t`                    |
| 查看完整配置           | `nginx -T`                    |
| 重载 Nginx         | `systemctl reload nginx`      |
| 安装 Keepalived    | `yum install keepalived -y`   |
| 查看 VIP           | `ip addr show dev ens33`      |
| 查看 Keepalived 状态 | `systemctl status keepalived` |

## 易错点

| 问题                         | 处理                                 |
| -------------------------- | ---------------------------------- |
| worker_connections 改很大仍不生效 | 检查文件描述符限制                          |
| gzip 压缩图片视频收益低             | 图片视频通常已压缩，不建议再压                    |
| ip_hash 不能彻底解决 session 问题  | 应优先做共享 session 或无状态认证              |
| upstream 后端日志只有代理 IP       | 配置 `X-Real-IP` / `X-Forwarded-For` |
| Keepalived 云主机不漂移          | 云环境通常不支持自建 VIP，改用云负载均衡             |
| 裂脑                         | 检查 VRRP 通信、防火墙、心跳链路、router_id、VRID |

## 关联条目

| 类型 | 条目                                                  |
| -- | --------------------------------------------------- |
| 前置 | [[Nginx 反向代理]]、[[TCP 三次握手]]、[[Linux 进程]]            |
| 核心 | [[Nginx 优化]]、[[Nginx 负载均衡]]、[[Keepalived]]、[[VRRP]] |
| 延伸 | [[云负载均衡]]、[[高可用架构]]、[[Nginx 日志分析]]                  |
