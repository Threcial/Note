---
title: DNS、CDN、HTTP HTTPS 与 Linux I/O 模型
type: knowledge
status: evergreen
source: day15.md
created: 2026-06-13
updated: 2026-06-13
tags:
  - DNS
  - CDN
  - HTTP
  - HTTPS
  - Linux
  - IO多路复用
  - epoll
  - 红黑树
aliases:
  - DAY15
  - DNS 与 CDN
  - HTTP HTTPS 原理
  - Linux IO 原理
Belongs to:
  - "[[网络基础]]"
  - "[[Web 服务]]"
  - "[[Linux 系统原理]]"
Related to:
  - "[[DAY06 - RPM、YUM、源码安装与网络基础]]"
  - "[[DAY07 - 网络命令、systemd 服务与进程管理]]"
  - "[[NAT]]"
  - "[[TCP 三次握手]]"
  - "[[文件描述符]]"
Has:
  - "[[DNS]]"
  - "[[A 记录]]"
  - "[[CNAME]]"
  - "[[TTL]]"
  - "[[CDN]]"
  - "[[源站]]"
  - "[[缓存刷新]]"
  - "[[HTTP]]"
  - "[[HTTPS]]"
  - "[[TLS]]"
  - "[[select]]"
  - "[[poll]]"
  - "[[epoll]]"
  - "[[红黑树]]"
  - "[[Page Cache]]"
---

# DNS、CDN、HTTP HTTPS 与 Linux I/O 模型

## 去重说明

前面的文件已经整理了 IP、路由、NAT、TCP 状态和网络命令。本条目保留更靠近 Web 访问链路的内容：DNS 解析、CDN、HTTP/HTTPS，以及 Linux 网络 I/O 多路复用和相关数据结构。

## 核心结论

| 主题 | 结论 |
| --- | --- |
| DNS | 把域名解析成 IP 或另一个域名，解析结果会被缓存 |
| A 记录 | 域名直接指向 IP |
| CNAME | 域名指向另一个域名，常用于 CDN |
| CDN | 用户访问 CDN 节点，CDN 节点再按缓存情况回源 |
| HTTP | 明文应用层协议，基于请求和响应 |
| HTTPS | HTTP over TLS，增加加密、身份认证和完整性保护 |
| I/O 多路复用 | 一个进程可以同时管理大量连接，核心机制包括 `select`、`poll`、`epoll` |

## DNS

DNS 用于把域名解析成访问目标。常见结果可能是：

```text
www.baidu.com -> 1.1.1.1
```

如果服务器故障，只是把 DNS 解析从坏机器 IP 改到新机器 IP，可能会受到缓存影响。因为客户端、本地 DNS、运营商 DNS、公共 DNS 都可能缓存旧结果。

更稳的故障切换方式之一是：让新机器接管旧机器的 IP。这样即使 DNS 缓存没有刷新，用户仍然会访问到可用机器。

## DNS 缓存与 TTL

DNS 记录通常有 TTL。TTL 决定解析结果可以被缓存多久。

| 位置 | 可能缓存 DNS |
| --- | --- |
| 浏览器 | 浏览器 DNS 缓存 |
| 操作系统 | 本机 DNS 缓存 |
| 本地 DNS | 路由器或运营商 DNS |
| 公共 DNS | 例如公共递归解析器 |
| 权威 DNS | 域名最终记录来源 |

TTL 调整后并不会让已经缓存的旧记录立刻全部失效，只能影响后续缓存周期。因此正式切换解析前，通常会提前降低 TTL。

## A 记录与 CNAME

### A 记录

A 记录把域名直接解析成 IP。

```text
www.abc.com -> 2.2.2.2
```

适合直接指向固定服务器 IP。

### CNAME

CNAME 把一个域名解析成另一个域名。

```text
xyz.abc.com -> www.taobao.com
cdn.abc.com -> www.a.shifen.com
```

CDN 场景中，厂商通常会提供一个 CNAME 地址。用户把自己的业务域名 CNAME 到 CDN 厂商域名，由 CDN 厂商继续调度到合适节点。

## DNS 记录关系

| 记录类型 | 作用 | 常见场景 |
| --- | --- | --- |
| `A` | 域名到 IPv4 | `www.example.com -> 1.2.3.4` |
| `AAAA` | 域名到 IPv6 | IPv6 访问 |
| `CNAME` | 域名到域名 | CDN、别名域名 |
| `MX` | 邮件服务器 | 企业邮箱 |
| `TXT` | 文本记录 | 域名验证、SPF、DKIM |
| `NS` | 指定权威 DNS | 域名托管 |
| `CAA` | 限制证书签发机构 | HTTPS 证书安全 |

## DNS 排查命令

```bash
nslookup www.example.com
dig www.example.com
dig www.example.com A
dig www.example.com CNAME
dig +trace www.example.com
```

常看字段：

| 字段 | 含义 |
| --- | --- |
| `ANSWER SECTION` | 最终解析答案 |
| `CNAME` | 是否经过别名 |
| `A` | 最终 IPv4 |
| `TTL` | 缓存时间 |
| `SERVER` | 当前使用的 DNS 服务器 |

## CDN

CDN 的核心是把内容缓存到离用户更近的节点，降低源站压力和访问延迟。

典型访问链路：

1. 用户访问 `cdn.abc.com`。
2. DNS 解析到 CDN 厂商提供的 CNAME。
3. CDN 调度系统返回合适节点 IP。
4. 用户访问 CDN 节点。
5. 节点命中缓存则直接返回。
6. 节点未命中缓存则回源站获取内容，再缓存并返回。

## 源站

源站是 CDN 回源访问的真实业务服务器。

源站配置可以是：

| 类型 | 说明 |
| --- | --- |
| IP 源站 | CDN 直接回源到指定 IP |
| 域名源站 | CDN 先解析源站域名，再回源 |
| 对象存储 | 静态资源源站 |

源站故障时，CDN 节点缓存命中的资源可能还能访问，未命中资源会失败。动态请求通常更依赖源站可用性。

## CDN 刷新与预热

| 操作 | 含义 |
| --- | --- |
| 刷新 | 删除 CDN 节点缓存，让下次访问重新回源 |
| 预热 | 主动让 CDN 节点提前拉取资源 |
| 回源 | CDN 节点向源站请求资源 |
| 缓存命中 | CDN 节点本地已有资源，直接返回 |
| 缓存未命中 | CDN 节点需要回源 |

修改源站资源后，如果 CDN 仍返回旧内容，通常需要刷新缓存。

## CDN 与 DNS 的关系

| 项目 | DNS | CDN |
| --- | --- | --- |
| 核心功能 | 域名解析 | 内容分发和缓存 |
| 是否传输内容 | 不传输网页内容 | 传输资源内容 |
| 常见记录 | A、CNAME | 通常依赖 CNAME 接入 |
| 故障影响 | 解析不到或解析到旧地址 | 资源过期、回源失败、节点故障 |

CDN 接入通常离不开 DNS，但 DNS 本身不是 CDN。

## HTTP

HTTP 是应用层协议，基于请求和响应工作。

请求基本结构：

```http
GET /index.html HTTP/1.1
Host: www.example.com
User-Agent: curl/8.0
```

响应基本结构：

```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1234
```

常见方法：

| 方法 | 用途 |
| --- | --- |
| `GET` | 获取资源 |
| `POST` | 提交数据 |
| `PUT` | 更新资源 |
| `DELETE` | 删除资源 |
| `HEAD` | 只获取响应头 |
| `OPTIONS` | 查询支持的方法 |

常见状态码：

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

HTTPS 是 HTTP over TLS。它在 HTTP 和 TCP 之间加入 TLS 层。

HTTPS 提供：

| 能力 | 说明 |
| --- | --- |
| 加密 | 防止传输内容被直接读取 |
| 身份认证 | 通过证书确认服务端身份 |
| 完整性 | 防止内容在传输中被篡改 |

典型关系：

```text
HTTP  -> 应用层明文协议
TLS   -> 加密与认证层
TCP   -> 可靠传输
IP    -> 网络寻址
```

HTTPS 排查常用命令：

```bash
curl -I https://www.example.com
openssl s_client -connect www.example.com:443 -servername www.example.com
```

`-servername` 用于指定 SNI。一个服务器上托管多个 HTTPS 站点时，SNI 很关键。

## Linux I/O 原理

Linux 程序读写文件或网络连接时，通常会通过系统调用进入内核。常见路径：

```text
用户进程 -> 系统调用 -> 内核 -> 文件系统/网络协议栈/设备驱动 -> 硬件
```

相关概念：

| 概念 | 说明 |
| --- | --- |
| 文件描述符 | 进程访问文件、socket、管道等资源的编号 |
| 用户态 | 应用程序运行空间 |
| 内核态 | 内核运行空间 |
| Page Cache | 内核用于缓存文件数据的内存区域 |
| 阻塞 I/O | 调用后等待数据就绪 |
| 非阻塞 I/O | 没有数据时立即返回 |
| I/O 多路复用 | 一个进程同时监听多个文件描述符 |

## select、poll、epoll

### select

`select` 可以监听多个文件描述符，但存在限制：

| 问题 | 说明 |
| --- | --- |
| 文件描述符数量限制 | 通常受 `FD_SETSIZE` 限制 |
| 每次都要拷贝集合 | 用户态和内核态之间传递成本较高 |
| 每次线性扫描 | 连接多时效率下降 |

### poll

`poll` 改进了 `select` 的文件描述符数量限制，但仍然需要线性扫描。

### epoll

`epoll` 是 Linux 下高并发网络服务常用机制。

特点：

| 特点 | 说明 |
| --- | --- |
| 不需要每次传入完整 fd 集合 | 先注册，再等待事件 |
| 事件驱动 | 只返回就绪事件 |
| 更适合大量连接 | 高并发服务器常用 |
| 支持水平触发和边缘触发 | LT/ET 模式 |

典型系统调用：

```text
epoll_create
epoll_ctl
epoll_wait
```

Nginx、Redis 等高并发网络程序常依赖 I/O 多路复用模型。

## 红黑树

红黑树是一种自平衡二叉搜索树，常见操作复杂度为 `O(log n)`。

特点：

| 特点 | 说明 |
| --- | --- |
| 有序 | 节点按键值排序 |
| 自平衡 | 通过旋转和变色维持近似平衡 |
| 查询效率稳定 | 插入、删除、查找通常为 `O(log n)` |

Linux 内核中很多结构会使用红黑树来管理有序对象。理解红黑树有助于理解一些内核数据结构和高性能组件的底层实现。

## HTTP、CDN、DNS 的完整访问链路

访问一个 HTTPS CDN 站点时，大致过程是：

1. 浏览器查询域名 DNS。
2. DNS 返回 CDN CNAME 或节点 IP。
3. 客户端与 CDN 节点建立 TCP 连接。
4. 双方进行 TLS 握手。
5. 客户端发送 HTTP 请求。
6. CDN 检查缓存。
7. 命中缓存则直接返回。
8. 未命中缓存则 CDN 回源。
9. 源站响应后，CDN 根据缓存策略保存并返回给用户。

## 易错点

| 问题 | 说明 |
| --- | --- |
| 修改 DNS 后不立即生效 | DNS 缓存和 TTL 会影响生效时间 |
| CNAME 不是 IP | CNAME 指向另一个域名 |
| CDN 返回旧内容 | 可能是节点缓存未刷新 |
| 源站正常但 CDN 异常 | 需要检查 CDN 配置、缓存、回源协议和回源 Host |
| HTTPS 证书正常但访问失败 | 还要检查 SNI、证书链、TLS 版本和源站端口 |
| `select/poll/epoll` 混为一谈 | 三者都是 I/O 多路复用，但性能模型不同 |
