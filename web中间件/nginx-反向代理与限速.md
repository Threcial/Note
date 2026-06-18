---
type: Practice
status: Organized
Belongs to: "[[Nginx]]"
Related to:
  - "[[nginx-虚拟主机与访问控制]]"
Has:
  - "[[Nginx 反向代理]]"
source: day16.md
created: 2026-06-14
updated: 2026-06-14
---

# Nginx 反向代理与限速

## 正向代理与反向代理

| 类型 | 使用者感知 | 典型场景 |
| --- | --- | --- |
| 正向代理 | 客户端知道代理服务器 | 客户端访问外部网络、VPN |
| 反向代理 | 客户端只知道代理入口 | Web 网关、隐藏后端、负载均衡 |

## 反向代理基础

```nginx
server {
    listen 80;
    server_name proxy.example.com;

    location / {
        proxy_pass http://192.168.31.23:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

后端日志记录真实 IP：

```nginx
log_format main '$http_x_real_ip|$http_x_forwarded_for - $remote_user ...';
```

## 限速

### 限制请求频率

```nginx
http {
    limit_req_zone $binary_remote_addr zone=req_limit:10m rate=1r/s;

    server {
        location / {
            limit_req zone=req_limit burst=5 nodelay;
        }
    }
}
```

| 参数 | 含义 |
| --- | --- |
| `rate=1r/s` | 平均每秒 1 个请求 |
| `burst=5` | 允许短时间突发 |
| `nodelay` | 超出限制立即返回错误 |

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

## 易错点

| 问题 | 处理 |
| --- | --- |
| 后端拿不到真实 IP | 反向代理要加 `X-Real-IP` 和 `X-Forwarded-For` |
