---
status: Organized
Belongs to: "[[Docker]]"
Related to:
  - "[[docker-命令行参考]]"
Has:
  - "[[Docker 数据卷]]"
  - "[[Docker 网络]]"
  - "[[docker volume]]"
  - "[[docker network]]"
source: ""
created: 2026-06-14
updated: 2026-06-14
type: Concept
---

# Docker 数据卷与网络

数据卷把数据从容器生命周期中分离出来。删除容器不会自动删除命名数据卷。

## Bind Mount

```bash
docker run -d -P -v /html:/usr/share/nginx/html nginx:latest
```

宿主机目录直接映射到容器目录。宿主机目录内容会覆盖容器目标目录视图。

## Named Volume

```bash
docker volume create web001
docker run -d -p 3333:80 --name nginx20 -v web001:/usr/share/nginx/html nginx
```

查看：

```bash
docker volume ls
docker volume inspect web001
docker inspect nginx20 | grep -A5 Mounts
```

删除：

```bash
docker volume rm web001
docker volume prune
```

`docker volume prune` 会删除未被容器使用的数据卷，执行前必须确认没有误删风险。

## 网络模式

安装 Docker 后会自动创建 `docker0` 网桥。默认 bridge 网络中，容器通过 veth pair 接入 docker0，再通过 NAT 访问外部。

| 网络模式 | 说明 | 参数 |
| --- | --- | --- |
| bridge | 默认模式，每个容器有独立 IP | `--network bridge` |
| host | 使用宿主机网络命名空间 | `--network host` |
| none | 不配置网络 | `--network none` |
| container | 与另一个容器共享网络 | `--network container:容器名` |

查看网络：

```bash
docker network ls
docker network inspect bridge
```

## 自定义网络

```bash
docker network create webnet
docker run -d --name nginx01 --network webnet nginx
docker run -d --name nginx02 --network webnet nginx
```

在用户自定义 bridge 网络里，容器可以直接用容器名互相解析。新版本 Docker 不再建议使用 `--link`，用户自定义网络内置 DNS 更干净。

### 容器内安装排查工具

```bash
apt-get update
apt-get install -y iputils-ping net-tools curl
```

Debian/Ubuntu 镜像更新慢时，可以换 `/etc/apt/sources.list` 或 deb822 源文件为国内镜像。

## 易错点

| 问题 | 处理 |
| --- | --- |
| 数据写在容器层 | 用 volume 或 bind mount 持久化 |
| 容器里 ping 不存在 | 安装 `iputils-ping` |
| 默认 bridge 容器名不互通 | 使用用户自定义 bridge 网络 |
