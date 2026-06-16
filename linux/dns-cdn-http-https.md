---
type: knowledge
status: evergreen
created: 2026-06-13
updated: 2026-06-13
related_to:
  - "[[网络基础]]"
  - "[[Web 服务]]"
has:
  - "[[DNS]]"
  - "[[CDN]]"
  - "[[HTTP]]"
  - "[[HTTPS]]"
  - "[[TLS]]"
_organized: true
---

# DNS、CDN、HTTP、HTTPS

## DNS

DNS 把域名解析成 IP。

### 记录类型

| 记录类型 | 作用 | 常见场景 |
| --- | --- | --- |
| `A` | 域名到 IPv4 | `www.example.com -> 1.2.3.4` |
| `AAAA` | 域名到 IPv6 | IPv6 访问 |
| `CNAME` | 域名到域名 | CDN、别名域名 |
| `MX` | 邮件服务器 | 企业邮箱 |
| `TXT` | 文本记录 | 域名验证、SPF |

### 缓存与 TTL

DNS 记录有 TTL，客户端、本地 DNS、公共 DNS 都可能缓存旧结果。正式切换解析前，通常会提前降低 TTL。

### 排查命令

```bash
nslookup www.example.com
dig www.example.com
dig +trace www.example.com
```

关键字段：`ANSWER SECTION`（最终答案）、`CNAME`、`A`、`TTL`、`SERVER`。

## CDN

### 访问链路

1. 用户访问域名 → DNS 解析到 CDN CNAME → 调度返回节点 IP → 用户访问 CDN 节点 → 命中缓存直接返回 / 未命中回源。

### 源站

| 类型 | 说明 |
| --- | --- |
| IP 源站 | CDN 直接回源到指定 IP |
| 域名源站 | CDN 先解析源站域名再回源 |
| 对象存储 | 静态资源源站 |

### 刷新与预热

| 操作 | 含义 |
| --- | --- |
| 刷新 | 删除 CDN 节点缓存 |
| 预热 | 主动让 CDN 节点提前拉取资源 |
| 回源 | CDN 节点向源站请求资源 |

修改源站资源后如果 CDN 仍返回旧内容，通常需要刷新缓存。

## HTTP

### 请求与响应

```http
GET /index.html HTTP/1.1
Host: www.example.com
```

```http
HTTP/1.1 200 OK
Content-Type: text/html
```

### 常见方法

| 方法 | 用途 |
| --- | --- |
| `GET` | 获取资源 |
| `POST` | 提交数据 |
| `PUT` | 更新资源 |
| `DELETE` | 删除资源 |
| `HEAD` | 只获取响应头 |

### 常见状态码

| 状态码 | 含义 |
| --- | --- |
| `200` | 成功 |
| `301/302` | 重定向 |
| `304` | 使用缓存 |
| `400` | 请求错误 |
| `401/403` | 未认证或无权限 |
| `404` | 资源不存在 |
| `500` | 服务端错误 |
| `502/503/504` | 网关、后端或超时问题 |

## HTTPS

HTTPS 是 HTTP over TLS，在 HTTP 和 TCP 之间加入 TLS 层。

提供：加密（防止内容被读取）、身份认证（证书确认服务端身份）、完整性（防止篡改）。

排查常用：

```bash
curl -I https://www.example.com
openssl s_client -connect www.example.com:443 -servername www.example.com
```

`-servername` 用于指定 SNI，一个服务器上托管多个 HTTPS 站点时很关键。

## 完整访问链路（HTTPS + CDN）

1. 浏览器查询域名 DNS
2. DNS 返回 CDN CNAME 或节点 IP
3. 客户端与 CDN 节点建立 TCP 连接
4. TLS 握手
5. 客户端发送 HTTP 请求
6. CDN 检查缓存 → 命中返回 / 未命中回源 → 源站响应 → CDN 保存并返回

## 易错点

| 问题 | 说明 |
| --- | --- |
| 修改 DNS 后不立即生效 | DNS 缓存和 TTL 会影响生效时间 |
| CNAME 不是 IP | CNAME 指向另一个域名 |
| CDN 返回旧内容 | 可能是节点缓存未刷新 |
| HTTPS 证书正常但访问失败 | 还要检查 SNI、证书链、TLS 版本 |
