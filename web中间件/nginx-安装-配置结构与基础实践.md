---
type: Practice
status: Organized
source: day16.md
created: 2026-06-14
updated: 2026-06-14
Belongs to:
  - "[[Nginx]]"
Related to:
  - "[[Linux 软件包管理]]"
  - "[[systemd 服务管理]]"
  - "[[HTTP 协议]]"
Has:
  - "[[Nginx 配置文件]]"
  - "[[Nginx 虚拟主机]]"
  - "[[Nginx 反向代理]]"
  - "[[Nginx 日志]]"
  - "[[Nginx 访问控制]]"
---
# Nginx 安装、配置结构与基础实践

Nginx 可以同时承担 Web 服务器、反向代理、负载均衡入口和静态资源服务角色。基础使用阶段最重要的不是堆配置项，而是先把几条主线理清：安装方式决定目录结构，目录结构决定配置路径，`server` 决定虚拟主机，`location` 决定 URI 命中后的处理逻辑，日志和验证命令决定排错效率。

Nginx 相比 Apache 更适合高并发静态资源和反向代理场景，核心原因在于事件驱动、异步非阻塞和 epoll 这类 I/O 模型。Apache 功能完整、生态成熟，但传统阻塞模型在高并发下资源消耗更明显。

## 关系总览

| 条目                         | 关系                     |
| -------------------------- | ---------------------- |
| Nginx                      | Web 服务、反向代理、负载均衡入口     |
| `server`                   | 一个虚拟主机配置块              |
| `location`                 | 一个 URI 或正则匹配处理块        |
| `root` / `alias`           | 决定 URI 映射到本地文件路径的方式    |
| `access_log` / `error_log` | 访问日志与错误日志，是排查请求问题的主要入口 |
| `proxy_pass`               | 把请求转交给后端服务，是反向代理配置的核心  |
| `limit_req` / `limit_conn` | 请求频率和并发连接限制            |

## 安装方式的选择

### 源码安装

源码安装适合需要自定义编译参数、指定安装目录、控制模块启用情况的场景。原始流程是：下载源码包、解压、创建运行用户、执行 `./configure`、`make`、`make install`。

```bash
yum install gcc zlib zlib-devel pcre-devel openssl-devel -y

wget http://nginx.org/download/nginx-1.19.5.tar.gz
tar xf nginx-1.19.5.tar.gz
cd nginx-1.19.5

useradd nginx -s /sbin/nologin -M

./configure \
  --user=nginx \
  --group=nginx \
  --prefix=/nginx \
  --with-http_ssl_module \
  --with-http_gzip_static_module

make
make install
```

源码安装后的常见目录：

| 目录                  | 作用        |
| ------------------- | --------- |
| `/nginx/conf`       | 配置文件目录    |
| `/nginx/html`       | 默认网站根目录   |
| `/nginx/logs`       | 日志目录      |
| `/nginx/sbin/nginx` | Nginx 主程序 |

源码安装的 Nginx 不会自动注册为 systemd 服务。如果要长期管理，最好补一个 service 文件，而不是每次手工执行 `/nginx/sbin/nginx`。

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

保存为：

```bash
vim /etc/systemd/system/nginx.service
systemctl daemon-reload
systemctl enable --now nginx
```

### yum 安装

yum 安装适合常规生产环境，优点是目录规范、自动创建 nginx 用户、自动集成 systemd。

```bash
yum install epel-release -y
yum install nginx -y
systemctl enable --now nginx
```

yum 安装后的常见目录：

| 目录                         | 作用      |
| -------------------------- | ------- |
| `/etc/nginx`               | 主配置与子配置 |
| `/etc/nginx/nginx.conf`    | 主配置文件   |
| `/etc/nginx/conf.d/*.conf` | 子配置文件   |
| `/usr/share/nginx/html`    | 默认站点目录  |
| `/var/log/nginx`           | 日志目录    |
| `/var/cache/nginx`         | 缓存目录    |

也可以配置 Nginx 官方 yum 源：

```ini
[nginx]
name=Nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1
```

保存为：

```bash
vim /etc/yum.repos.d/nginx.repo
yum install nginx -y
```

## 启动、验证与排错入口

源码安装常用命令：

```bash
/nginx/sbin/nginx
/nginx/sbin/nginx -t
/nginx/sbin/nginx -s reload
/nginx/sbin/nginx -s stop
/nginx/sbin/nginx -s quit
/nginx/sbin/nginx -s reopen
```

yum 安装常用命令：

```bash
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx
systemctl status nginx
systemctl enable nginx
```

验证端口：

```bash
ss -lntp | grep ':80'
lsof -i:80
ps -ef | grep nginx
curl -I http://127.0.0.1/
```

现代排查中优先使用 `ss` 替代 `netstat`：

```bash
ss -lntp
ss -antp | grep nginx
```

查看完整生效配置：

```bash
nginx -T
```

`nginx -T` 会输出主配置和所有 `include` 进来的配置，比只看 `nginx.conf` 更适合排查线上配置。

## Nginx 配置文件结构

典型结构如下：

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

配置块关系：

| 配置块      | 作用                       |
| -------- | ------------------------ |
| main     | 全局配置，如运行用户、worker 数、错误日志 |
| events   | 连接处理模型和连接数               |
| http     | HTTP 服务全局配置              |
| server   | 虚拟主机                     |
| location | URI 匹配规则与请求处理            |

服务三要素：

| 要素 | 示例                                  |
| -- | ----------------------------------- |
| 地址 | `0.0.0.0`、`127.0.0.1`、具体网卡 IP       |
| 端口 | HTTP 80、HTTPS 443、SSH 22、MySQL 3306 |
| 协议 | HTTP、HTTPS、TCP、UDP                  |

## root 与 alias

`root` 是把 URI 拼接到指定目录后面。

```nginx
location /images/ {
    root /data/www;
}
```

访问 `/images/a.png` 时，实际文件路径是：

```text
/data/www/images/a.png
```

`alias` 是用指定目录替换命中的 URI 前缀。

```nginx
location /images/ {
    alias /data/pic/;
}
```

访问 `/images/a.png` 时，实际文件路径是：

```text
/data/pic/a.png
```

`alias` 最容易踩坑的是结尾斜杠。`location /images/` 配合 `alias /data/pic/` 最清晰，避免 URI 拼接异常。

## 访问控制

基于来源 IP 限制访问：

```nginx
location /web1/ {
    allow 127.0.0.1;
    allow 192.168.31.0/24;
    deny all;

    root /usr/share/nginx/html;
    index index.html;
}
```

规则按顺序匹配。先允许具体来源，再 `deny all`。如果顺序反了，后面的 allow 不会有机会生效。

验证：

```bash
curl -I http://127.0.0.1/web1/
curl -I http://192.168.31.20/web1/
```

## HTTP 基本认证

安装工具：

```bash
yum install httpd-tools -y
```

生成密码文件：

```bash
mkdir -p /etc/nginx/passwd
htpasswd -c /etc/nginx/passwd/htpasswd xwx
htpasswd -m /etc/nginx/passwd/htpasswd xb
```

Nginx 配置：

```nginx
location /private/ {
    auth_basic "private area";
    auth_basic_user_file /etc/nginx/passwd/htpasswd;

    root /usr/share/nginx/html;
    index index.html;
}
```

测试配置并重载：

```bash
nginx -t
systemctl reload nginx
```

## 防盗链

防盗链基于 HTTP 请求头中的 `Referer`。常见配置是只允许无 Referer、被隐藏的 Referer、本站域名访问图片资源。

```nginx
location ~* \.(png|jpg|jpeg|gif|webp)$ {
    valid_referers none blocked server_names *.example.com example.com;

    if ($invalid_referer) {
        return 403;
    }

    root /usr/share/nginx/html;
}
```

`Referer` 可以伪造，所以防盗链不是强安全机制，只适合降低普通盗链消耗。如果资源涉及权限，应使用鉴权 URL、签名 URL 或对象存储/CDN 的防盗链策略。

## 虚拟主机

### 基于 IP

```nginx
server {
    listen 192.168.0.129:80;
    server_name _;
    root /usr/share/nginx/html/web1;
    index index.html;
}

server {
    listen 192.168.0.130:80;
    server_name _;
    root /usr/share/nginx/html/web2;
    index index.html;
}
```

旧命令：

```bash
ifconfig ens33:1 192.168.0.130 up
```

更常用的 `ip` 写法：

```bash
ip addr add 192.168.0.130/24 dev ens33
ip addr show dev ens33
ip addr del 192.168.0.130/24 dev ens33
```

CentOS 7 中如果要永久配置多 IP，可以写入网卡配置文件，也可以由 NetworkManager 管理，具体取决于当前环境是否启用 NetworkManager。

### 基于端口

```nginx
server {
    listen 80;
    root /usr/share/nginx/html/web1;
    index index.html;
}

server {
    listen 8080;
    root /usr/share/nginx/html/web2;
    index index.html;
}
```

访问方式：

```bash
curl http://192.168.0.129/
curl http://192.168.0.129:8080/
```

### 基于域名

```nginx
server {
    listen 80;
    server_name www.abc.com;
    root /usr/share/nginx/html/web1;
    index index.html;
}

server {
    listen 80;
    server_name www.aaa.com;
    root /usr/share/nginx/html/web2;
    index index.html;
}
```

本地测试可临时写 hosts：

```bash
vim /etc/hosts
```

```text
192.168.0.129 www.abc.com www.aaa.com
```

## 日志管理

自定义日志格式必须写在 `http` 块中：

```nginx
log_format web1 '[$time_local] $remote_addr $status "$request"';
log_format web2 '[$time_local] $remote_addr $status "$http_user_agent"';
```

虚拟主机调用日志格式：

```nginx
server {
    listen 80;
    server_name web1.example.com;
    access_log /var/log/nginx/web1.access.log web1;
}

server {
    listen 80;
    server_name web2.example.com;
    access_log /var/log/nginx/web2.access.log web2;
}
```

常用日志变量：

| 变量                      | 含义          |
| ----------------------- | ----------- |
| `$remote_addr`          | 客户端 IP      |
| `$http_x_forwarded_for` | 代理链中的客户端 IP |
| `$remote_user`          | 认证用户        |
| `$time_local`           | 本地时间        |
| `$request`              | 请求方法、URI、协议 |
| `$status`               | 响应状态码       |
| `$body_bytes_sent`      | 响应体字节数      |
| `$http_referer`         | 来源页面        |
| `$http_user_agent`      | 浏览器或客户端信息   |

日志切割后需要让 Nginx 重新打开日志文件：

```bash
nginx -s reopen
```

## include 配置拆分

```nginx
include /etc/nginx/conf.d/*.conf;
```

把多个站点拆成多个 `.conf` 文件后，主配置更清晰。排查时要注意：真正生效的配置不一定只在 `nginx.conf` 里，`nginx -T` 更可靠。

## 正向代理与反向代理

| 类型   | 使用者感知              | 典型场景                  |
| ---- | ------------------ | --------------------- |
| 正向代理 | 客户端知道代理服务器         | 客户端访问外部网络、VPN、代理出口    |
| 反向代理 | 客户端只知道代理入口，不知道真实后端 | Web 网关、隐藏后端、统一入口、负载均衡 |

反向代理基础配置：

```nginx
server {
    listen 80;
    server_name proxy.example.com;

    location / {
        proxy_pass http://192.168.31.23:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

后端日志想记录真实客户端 IP，需要日志格式引用 `X-Real-IP` 或 `X-Forwarded-For`：

```nginx
log_format main '$http_x_real_ip|$http_x_forwarded_for - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" "$http_user_agent"';
```

## 限速

### 限制请求频率

```nginx
http {
    limit_req_zone $binary_remote_addr zone=req_limit:10m rate=1r/s;

    server {
        location / {
            limit_req zone=req_limit burst=5 nodelay;
            root /usr/share/nginx/html;
        }
    }
}
```

| 参数                    | 含义               |
| --------------------- | ---------------- |
| `$binary_remote_addr` | 用客户端 IP 作为限制键    |
| `zone=req_limit:10m`  | 创建 10M 共享内存区     |
| `rate=1r/s`           | 平均每秒 1 个请求       |
| `burst=5`             | 允许短时间突发队列        |
| `nodelay`             | 超出限制立即返回错误，不排队等待 |

### 限制并发连接和下载速度

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    server {
        location /download/ {
            limit_conn conn_limit 1;
            limit_rate 10k;
            root /data;
        }
    }
}
```

## 命令速查

| 目的     | 命令                                           |
| ------ | -------------------------------------------- |
| 检查配置   | `nginx -t`                                   |
| 输出完整配置 | `nginx -T`                                   |
| 重载配置   | `nginx -s reload` / `systemctl reload nginx` |
| 重新打开日志 | `nginx -s reopen`                            |
| 查看监听端口 | `ss -lntp                                    |
| 查看进程   | `ps -ef                                      |
| 测试响应头  | `curl -I http://127.0.0.1/`                  |
| 查看访问日志 | `tail -f /var/log/nginx/access.log`          |
| 查看错误日志 | `tail -f /var/log/nginx/error.log`           |

## 易错点

| 问题                    | 处理                                       |
| --------------------- | ---------------------------------------- |
| 源码安装后 `systemctl` 管不了 | 需要手工创建 systemd unit                      |
| 改了配置不生效               | 先 `nginx -t`，再 reload；确认配置被 `include` 引入 |
| 访问 403                | 检查文件权限、SELinux、防火墙、Nginx `allow/deny`    |
| 访问 404                | 检查 `root` / `alias` 的实际路径拼接              |
| 后端拿不到真实 IP            | 反向代理要加 `X-Real-IP` 和 `X-Forwarded-For`   |
| 日志切割后不继续写新文件          | 执行 `nginx -s reopen`                     |
| `ifconfig` 不存在        | 安装 `net-tools`，或改用 `ip addr`             |

## 关联条目

| 类型 | 条目                                                     |
| -- | ------------------------------------------------------ |
| 前置 | [[Linux 软件包管理]]、[[源码安装]]、[[systemd 服务管理]]、[[Linux 日志]] |
| 核心 | [[Nginx 配置文件]]、[[Nginx 虚拟主机]]、[[Nginx 反向代理]]           |
| 延伸 | [[Nginx URL 重写]]、[[Nginx 优化]]、[[负载均衡]]、[[Keepalived]]  |
