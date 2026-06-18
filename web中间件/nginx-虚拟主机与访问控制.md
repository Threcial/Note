---
type: Practice
status: Organized
Belongs to: "[[Nginx]]"
Related to:
  - "[[nginx-安装与配置结构]]"
Has:
  - "[[Nginx 虚拟主机]]"
  - "[[Nginx 访问控制]]"
  - "[[Nginx 日志]]"
source: day16.md
created: 2026-06-14
updated: 2026-06-14
---

# Nginx 虚拟主机与访问控制

## 虚拟主机

### 基于 IP

```nginx
server {
    listen 192.168.0.129:80;
    root /usr/share/nginx/html/web1;
    index index.html;
}

server {
    listen 192.168.0.130:80;
    root /usr/share/nginx/html/web2;
    index index.html;
}
```

添加虚拟 IP：

```bash
ip addr add 192.168.0.130/24 dev ens33
```

### 基于端口

```nginx
server {
    listen 80;
    root /usr/share/nginx/html/web1;
}

server {
    listen 8080;
    root /usr/share/nginx/html/web2;
}
```

### 基于域名

```nginx
server {
    listen 80;
    server_name www.abc.com;
    root /usr/share/nginx/html/web1;
}

server {
    listen 80;
    server_name www.aaa.com;
    root /usr/share/nginx/html/web2;
}
```

## root 与 alias

`root` 把 URI 拼接到指定目录后面：

```nginx
location /images/ {
    root /data/www;
}
# /images/a.png → /data/www/images/a.png
```

`alias` 用指定目录替换 URI 前缀：

```nginx
location /images/ {
    alias /data/pic/;
}
# /images/a.png → /data/pic/a.png
```

## 访问控制

基于来源 IP：

```nginx
location /web1/ {
    allow 127.0.0.1;
    allow 192.168.31.0/24;
    deny all;
    root /usr/share/nginx/html;
}
```

规则按顺序匹配，先允许再拒绝。

## HTTP 基本认证

```bash
yum install httpd-tools -y
htpasswd -c /etc/nginx/passwd/htpasswd xwx
```

```nginx
location /private/ {
    auth_basic "private area";
    auth_basic_user_file /etc/nginx/passwd/htpasswd;
    root /usr/share/nginx/html;
}
```

## 防盗链

```nginx
location ~* \.(png|jpg|jpeg|gif|webp)$ {
    valid_referers none blocked server_names *.example.com;
    if ($invalid_referer) { return 403; }
    root /usr/share/nginx/html;
}
```

防盗链不是强安全机制，只适合降低普通盗链消耗。

## 日志管理

自定义日志格式必须写在 `http` 块中：

```nginx
log_format web1 '[$time_local] $remote_addr $status "$request"';
```

虚拟主机调用：

```nginx
server {
    listen 80;
    server_name web1.example.com;
    access_log /var/log/nginx/web1.access.log web1;
}
```

日志切割后重新打开：

```bash
nginx -s reopen
```

## 易错点

| 问题 | 处理 |
| --- | --- |
| 访问 403 | 检查文件权限、SELinux、防火墙、`allow/deny` |
| 访问 404 | 检查 `root` / `alias` 路径拼接 |
| 日志切割后不续写 | 执行 `nginx -s reopen` |
