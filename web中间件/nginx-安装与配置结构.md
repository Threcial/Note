---
type: Practice
status: Organized
Belongs to: "[[Nginx]]"
Related to:
  - "[[Linux 软件包管理]]"
  - "[[systemd 服务管理]]"
Has:
  - "[[Nginx 配置文件]]"
source: day16.md
created: 2026-06-14
updated: 2026-06-14
---

# Nginx 安装与配置结构

## 源码安装

```bash
yum install gcc zlib zlib-devel pcre-devel openssl-devel -y

wget http://nginx.org/download/nginx-1.19.5.tar.gz
tar xf nginx-1.19.5.tar.gz
cd nginx-1.19.5

./configure \
  --user=nginx --group=nginx \
  --prefix=/nginx \
  --with-http_ssl_module \
  --with-http_gzip_static_module

make && make install
```

源码安装后目录结构：

| 目录 | 作用 |
| --- | --- |
| `/nginx/conf` | 配置文件目录 |
| `/nginx/html` | 默认网站根目录 |
| `/nginx/logs` | 日志目录 |
| `/nginx/sbin/nginx` | 主程序 |

源码安装的 Nginx 不会自动注册 systemd，需手动创建 service：

```ini
[Unit]
Description=Nginx Web Server
After=network.target

[Service]
Type=forking
ExecStart=/nginx/sbin/nginx
ExecReload=/nginx/sbin/nginx -s reload
ExecStop=/nginx/sbin/nginx -s quit
PIDFile=/nginx/logs/nginx.pid

[Install]
WantedBy=multi-user.target
```

## yum 安装

```bash
yum install epel-release -y
yum install nginx -y
systemctl enable --now nginx
```

yum 安装后目录：

| 目录 | 作用 |
| --- | --- |
| `/etc/nginx` | 主配置与子配置 |
| `/etc/nginx/nginx.conf` | 主配置文件 |
| `/etc/nginx/conf.d/*.conf` | 子配置文件 |
| `/usr/share/nginx/html` | 默认站点目录 |
| `/var/log/nginx` | 日志目录 |

## 启动与验证

```bash
nginx -t            # 检查配置
nginx -T            # 输出全部配置
nginx -s reload     # 重载配置
nginx -s reopen     # 重新打开日志

systemctl start nginx
systemctl reload nginx
systemctl status nginx
```

## 配置文件结构

```nginx
user nginx;
worker_processes auto;

error_log /var/log/nginx/error.log warn;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    keepalive_timeout 65;

    include /etc/nginx/conf.d/*.conf;
}
```

| 配置块 | 作用 |
| --- | --- |
| main | 全局配置（用户、worker、错误日志） |
| events | 连接处理模型和连接数 |
| http | HTTP 服务全局配置 |
| server | 虚拟主机 |
| location | URI 匹配规则与请求处理 |

## include 配置拆分

```nginx
include /etc/nginx/conf.d/*.conf;
```

排查配置时用 `nginx -T`，比只看 `nginx.conf` 更可靠。

## 易错点

| 问题 | 处理 |
| --- | --- |
| 源码安装后 `systemctl` 管不了 | 需要手工创建 systemd unit |
| 改了配置不生效 | 先 `nginx -t`，再 reload |
| `ifconfig` 不存在 | 安装 `net-tools` 或改用 `ip addr` |
