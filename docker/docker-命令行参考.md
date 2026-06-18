---
type: Command
status: Organized
Belongs to: "[[Docker]]"
Related to:
  - "[[docker-基础概念]]"
  - "[[docker-数据卷与网络]]"
Has:
  - "[[docker run]]"
  - "[[docker exec]]"
  - "[[docker ps]]"
  - "[[docker logs]]"
  - "[[docker commit]]"
  - "[[docker save]]"
  - "[[docker load]]"
  - "[[docker cp]]"
source: ""
created: 2026-06-14
updated: 2026-06-14
---

# Docker 命令行参考

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
docker history nginx --format "table {.ID}\t{.CreatedBy}" --no-trunc
```

## 容器运行

```bash
docker run nginx:latest

docker run -d -p 1192:80 nginx:latest

docker run -d -p 1721:80 --name nginx01 nginx:latest

docker run -it nginx:latest bash
```

常用参数：

| 参数 | 作用 |
| --- | --- |
| `-d` | 后台运行 |
| `-p 宿主机端口:容器端口` | 端口映射 |
| `-P` | 随机映射镜像声明端口 |
| `--name` | 指定容器名 |
| `-it` | 交互终端 |
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

已启动容器的自启配置：

```bash
docker update --restart=always 容器ID
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

排查容器启动失败：`docker ps -a` → `docker logs` → `docker inspect`。

## commit、save、load

```bash
docker commit -m "ngx-test" -a "xwx" nginx01 ngx-01:latest

docker save ngx-01:latest -o ngx-01.tar

docker load -i ngx-01.tar
```

`docker commit` 适合实验和临时固化，不适合生产镜像构建。生产镜像应该用 Dockerfile。

## 镜像分层

Docker 镜像由多层只读层叠加形成。公共层可以复用，减少下载和存储。Dockerfile 中每条构建指令通常都会产生一层，构建时要把相关命令合并，减少层数和无效缓存。

```dockerfile
RUN yum install -y make gcc pcre-devel zlib-devel \
    && yum clean all \
    && rm -rf /var/cache/yum
```

## 部署常见服务

### MySQL

```bash
docker run -d --name mysql57 \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD='复杂密码' \
  -v mysql57_data:/var/lib/mysql \
  mysql:5.7.38
```

MySQL 容器首次启动必须设置 root 密码或初始化相关环境变量，否则容器会退出。

### Redis

```bash
docker run -d --name redis \
  -p 6379:6379 \
  -v redis_data:/data \
  redis:5.0.7 redis-server --appendonly yes
```

### Nginx

```bash
docker run -d --name nginx \
  -p 80:80 \
  -v /html:/usr/share/nginx/html \
  nginx:latest
```

## 易错点

| 问题 | 处理 |
| --- | --- |
| 容器启动后马上退出 | 主进程退出了，需要前台进程保持运行 |
| 删除镜像失败 | 先删除依赖该镜像的容器 |
| `docker cp` 当长期同步方案 | 改用 volume |
| 镜像拉取慢 | 更换 registry mirrors 或私有仓库 |
