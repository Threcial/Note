---
title: Docker 基础、镜像、容器、数据卷与网络
type: knowledge
status: evergreen
source: day26_docker.md
created: 2026-06-14
updated: 2026-06-14
tags:
  - Docker
  - 容器
  - 镜像
  - 数据卷
  - Docker网络
  - CentOS7
  - 运维基础
aliases:
  - DAY26
  - Docker 基础
  - Docker 镜像
  - Docker 容器
  - Docker 数据卷
  - Docker 网络
Belongs to:
  - "[[容器技术]]"
  - "[[Linux 运维基础]]"
  - "[[Docker]]"
Related to:
  - "[[Linux Namespace]]"
  - "[[cgroups]]"
  - "[[OverlayFS]]"
  - "[[Nginx]]"
  - "[[MySQL]]"
  - "[[Redis]]"
Has:
  - "[[Docker Image]]"
  - "[[Docker Container]]"
  - "[[Docker Repository]]"
  - "[[docker run]]"
  - "[[docker exec]]"
  - "[[docker logs]]"
  - "[[docker volume]]"
  - "[[docker network]]"
  - "[[docker0]]"
  - "[[Docker Bridge 网络]]"
_organized: true
---
# Docker 基础、镜像、容器、数据卷与网络

Docker 把应用和运行环境打包成镜像，再把镜像运行成容器。镜像是静态模板，容器是运行中的进程。容器不是虚拟机，它共享宿主机内核，通过 namespace、cgroups、联合文件系统实现隔离和资源控制。

## 关系整理

| 对象            | 作用               |
| ------------- | ---------------- |
| 镜像 Image      | 只读模板，包含应用和运行依赖   |
| 容器 Container  | 镜像运行后的进程实体       |
| 仓库 Repository | 存放镜像的远程或本地仓库     |
| 数据卷 Volume    | 把数据从容器生命周期中拆出来   |
| 网络 Network    | 负责容器与容器、容器与宿主机通信 |
| Dockerfile    | 自动构建镜像的描述文件      |
| Compose       | 多容器编排工具          |

## Docker 与虚拟机

| 对比项  | 容器              | 虚拟机          |
| ---- | --------------- | ------------ |
| 内核   | 共享宿主机内核         | 每台虚拟机有完整系统内核 |
| 启动速度 | 秒级或更快           | 通常分钟级        |
| 资源占用 | 较低              | 较高           |
| 隔离层次 | 操作系统进程级隔离       | 硬件虚拟化隔离      |
| 适用场景 | 应用交付、环境一致性、弹性部署 | 强隔离、多系统运行    |

Docker 的核心收益是环境一致、迁移方便、启动快、资源利用率高。它不适合被理解成“小型虚拟机”，更准确的理解是“带隔离环境的进程”。

## 安装 Docker

CentOS 7 安装示例：

```bash
yum remove -y docker*
yum install -y yum-utils
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io
systemctl start docker
systemctl enable docker
docker version
docker info
```

如果 compose 插件依赖安装失败，可以手动下载对应 rpm 安装。

镜像加速配置：

```bash
mkdir -p /etc/docker
cat >/etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1panel.live/",
    "https://dockerhub.icu",
    "https://docker.awsl9527.cn",
    "https://dockerpull.com"
  ]
}
EOF
systemctl daemon-reload
systemctl restart docker
```

镜像站可用性会变化，拉取失败时需要更换源或使用自己的镜像加速地址。

## 镜像操作

```bash
docker images
docker image ls
docker image ls -q

docker pull redis:6.0
docker pull nginx

docker rmi nginx:latest
docker image rm redis:6.0

docker search redis
```

`docker search` 只能做粗略搜索，不适合查版本。具体 tag 更适合在 Docker Hub 或镜像仓库页面查。

查看镜像分层：

```bash
docker history nginx
docker history nginx --format "table {.ID}	{.CreatedBy}" --no-trunc
```

## 容器运行

```bash
docker run nginx:latest

docker run -d -p 1192:80 nginx:latest

docker run -d -p 1721:80 --name nginx01 nginx:latest

docker run -it nginx:latest bash
```

常用参数：

| 参数                 | 作用                     |
| ------------------ | ---------------------- |
| `-d`               | 后台运行                   |
| `-p 宿主机端口:容器端口`    | 端口映射                   |
| `-P`               | 随机映射镜像声明端口             |
| `--name`           | 指定容器名                  |
| `-it`              | 交互终端                   |
| `--restart=always` | 容器异常退出或 Docker 重启后自动拉起 |

容器启动后必须有前台进程存在。主进程退出，容器就退出。

## 查看、启停、删除容器

```bash
docker ps
docker ps -a
docker ps -aq

docker start nginx01
docker stop nginx01
docker restart nginx01
docker kill nginx01

docker rm nginx01
docker rm -f nginx01
docker rm -f $(docker ps -qa)
```

已经启动的容器配置自启：

```bash
docker update --restart=always 容器ID
```

取消自启：

```bash
docker update --restart=no 容器ID
```

## 进入容器与执行命令

```bash
docker exec -it nginx01 bash
docker exec nginx01 nginx -t
docker top nginx01
```

`docker exec` 是进入已运行容器的常用方式。`docker run -it image bash` 是新建一个容器并进入，退出后容器通常也会退出。

## 宿主机与容器复制文件

```bash
# 容器到宿主机
docker cp nginx01:/usr/share/nginx/html/index.html /root/

# 宿主机到容器
docker cp /root/test.html nginx01:/usr/share/nginx/html/
```

`docker cp` 适合临时拷贝。需要长期同步代码、配置或数据时，应使用 volume 或 bind mount。

## 查看容器日志和详情

```bash
docker logs nginx01
docker logs -f nginx01
docker logs --tail 20 nginx01
docker logs -t --tail 100 nginx01

docker inspect nginx01 | less
```

排查容器启动失败时，先看：

```bash
docker ps -a
docker logs 容器名
docker inspect 容器名
```

## commit、save、load

把容器提交为镜像：

```bash
docker commit -m "ngx-test" -a "xwx" nginx01 ngx-01:latest
```

保存镜像为 tar：

```bash
docker save ngx-01:latest -o ngx-01.tar
```

导入镜像：

```bash
docker load -i ngx-01.tar
```

`docker commit` 适合实验和临时固化，不适合生产镜像构建。生产镜像应该用 Dockerfile 保证可追溯、可重复构建。

## 镜像分层

Docker 镜像由多层只读层叠加形成。公共层可以复用，减少下载和存储。Dockerfile 中每条构建指令通常都会产生一层，因此构建时要把相关命令合并，减少层数和无效缓存。

示例：

```dockerfile
RUN yum install -y make gcc pcre-devel zlib-devel     && yum clean all     && rm -rf /var/cache/yum
```

## 数据卷

数据卷把数据从容器生命周期中分离出来。删除容器不会自动删除命名数据卷。

### bind mount

```bash
docker run -d -P -v /html:/usr/share/nginx/html nginx:latest
```

宿主机目录直接映射到容器目录。注意：宿主机目录内容会覆盖容器目标目录视图。

### named volume

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

## Docker 网络

安装 Docker 后会自动创建 `docker0` 网桥。默认 bridge 网络中，容器通过 veth pair 接入 docker0，再通过 NAT 访问外部。

| 网络模式      | 说明              | 参数                        |
| --------- | --------------- | ------------------------- |
| bridge    | 默认模式，每个容器有独立 IP | `--network bridge`        |
| host      | 使用宿主机网络命名空间     | `--network host`          |
| none      | 不配置网络           | `--network none`          |
| container | 与另一个容器共享网络      | `--network container:容器名` |

查看网络：

```bash
docker network ls
docker network inspect bridge
```

创建用户自定义网络：

```bash
docker network create webnet
docker run -d --name nginx01 --network webnet nginx
docker run -d --name nginx02 --network webnet nginx
```

在用户自定义 bridge 网络里，容器可以直接用容器名互相解析。新版本 Docker 不再建议使用 `--link`，用户自定义网络内置 DNS 更干净。

进入容器安装排查工具：

```bash
apt-get update
apt-get install -y iputils-ping net-tools curl
```

Debian/Ubuntu 镜像更新慢时，可以换 `/etc/apt/sources.list` 或 deb822 源文件为国内镜像。

## 部署常见服务

### MySQL

```bash
docker run -d --name mysql57   -p 3306:3306   -e MYSQL_ROOT_PASSWORD='复杂密码'   -v mysql57_data:/var/lib/mysql   mysql:5.7.38
```

MySQL 容器首次启动必须设置 root 密码或初始化相关环境变量，否则容器会退出。

### Redis

```bash
docker run -d --name redis   -p 6379:6379   -v redis_data:/data   redis:5.0.7 redis-server --appendonly yes
```

### Nginx

```bash
docker run -d --name nginx   -p 80:80   -v /html:/usr/share/nginx/html   nginx:latest
```

## 运维排查命令补充

```bash
# 目录大小
du -h --max-depth=1

# CPU 前 10 进程
ps aux | grep -v 'PID' | sort -rn -k3 | head -10

# 内存前 10 进程
ps aux | grep -v 'PID' | sort -rn -k4 | head -10

# 查看进程线程数
for pid in $(ps -ef | awk '{print $2}'); do
  [ -r /proc/$pid/status ] && awk '/Threads/ {print "PID='"$pid"'", $0}' /proc/$pid/status
done
```

## 易错点

| 问题                  | 处理                        |
| ------------------- | ------------------------- |
| 容器启动后马上退出           | 主进程退出了，需要前台进程保持运行         |
| 删除镜像失败              | 先删除依赖该镜像的容器               |
| 容器里 ping 不存在        | 安装 `iputils-ping`         |
| 默认 bridge 容器名不互通    | 使用用户自定义 bridge 网络         |
| 数据写在容器层             | 用 volume 或 bind mount 持久化 |
| `docker cp` 当长期同步方案 | 改用 volume                 |
| 镜像拉取慢               | 更换 registry mirrors 或私有仓库 |
