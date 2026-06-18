---
type: Reference
status: Organized
Belongs to: "[[Nginx]]"
Related to:
  - "[[nginx-反向代理与限速]]"
Has:
  - "[[upstream]]"
  - "[[gzip]]"
source: day18.md
created: 2026-06-14
updated: 2026-06-14
---

# Nginx 优化与负载均衡

## 性能优化

### worker

```nginx
worker_processes auto;
```

### 连接数

```nginx
events {
    worker_connections 4096;
}
```

理论上限 = worker_processes × worker_connections，实际受文件描述符限制。

查看限制：

```bash
ulimit -n
```

systemd 管理的 Nginx 还需设置：

```ini
[Service]
LimitNOFILE=65535
```

### 上传大小

```nginx
client_max_body_size 100m;
```

### 错误日志级别

```nginx
error_log /var/log/nginx/error.log error;
```

| 级别 | 说明 |
| --- | --- |
| `warn` | 警告及以上 |
| `error` | 错误及以上 |
| `crit` | 严重错误 |

### 长连接

```nginx
keepalive_timeout 15;
keepalive_requests 8192;
```

### sendfile

```nginx
sendfile on;
```

减少用户态和内核态之间的数据拷贝。

### gzip

```nginx
gzip on;
gzip_comp_level 6;
gzip_types text/plain text/css application/javascript application/json;
```

压缩等级 `4` 到 `6` 常见。图片视频已压缩的不适合再压。

### 隐藏版本号

```nginx
server_tokens off;
```

## 负载均衡

### upstream 定义

```nginx
upstream web {
    server 192.168.0.120;
    server 192.168.0.121;
    server 192.168.0.122;
}
```

### 权重

```nginx
upstream web {
    server 192.168.0.120 weight=10;
    server 192.168.0.121 weight=3;
    server 192.168.0.122;
}
```

### ip_hash

```nginx
upstream web {
    ip_hash;
    server 192.168.0.120;
    server 192.168.0.121;
}
```

让同一客户端尽量落到同一后端。

### 后端状态

| 参数 | 含义 |
| --- | --- |
| `down` | 不参与请求处理 |
| `backup` | 其他后端不可用时才参与 |
| `max_fails` | 失败次数阈值 |
| `fail_timeout` | 失败统计窗口 |

### 分发方式

基于域名：

```nginx
server { server_name www.abc.com; location / { proxy_pass http://web1; } }
server { server_name www.def.com; location / { proxy_pass http://web2; } }
```

基于目录：

```nginx
location /api/ { proxy_pass http://api_backend; }
location / { proxy_pass http://web_backend; }
```

基于浏览器 UA（推荐用 `map`）：

```nginx
map $http_user_agent $backend {
    default chrome;
    ~*elinks elinks;
}
```

## 连接数观测

```bash
ss -antp | grep nginx
ss -antp | grep nginx | grep ESTAB | wc -l
```

## 易错点

| 问题 | 处理 |
| --- | --- |
| worker_connections 改大仍不生效 | 检查文件描述符限制 |
| gzip 压缩图片视频收益低 | 图片视频已压缩，不建议再压 |
| ip_hash 不能解决 session | 应做共享 session 或无状态认证 |
| upstream 后端日志只有代理 IP | 配置 `X-Real-IP` / `X-Forwarded-For` |
