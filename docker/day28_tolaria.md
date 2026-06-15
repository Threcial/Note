---
title: Dockerfile、Docker Compose 与 Kubernetes 入门关系
type: knowledge
status: evergreen
source: day28.md
created: 2026-06-14
updated: 2026-06-14
tags:
  - Docker
  - Dockerfile
  - Docker Compose
  - YAML
  - 容器编排
  - Kubernetes
aliases:
  - DAY28
  - Dockerfile
  - Docker Compose
  - Compose
  - K8s 入门
Belongs to:
  - "[[Docker]]"
  - "[[容器技术]]"
  - "[[容器编排]]"
Related to:
  - "[[Docker 镜像]]"
  - "[[Docker 容器]]"
  - "[[Docker 网络]]"
  - "[[Docker 数据卷]]"
  - "[[YAML]]"
  - "[[Kubernetes]]"
Has:
  - "[[FROM]]"
  - "[[RUN]]"
  - "[[COPY]]"
  - "[[ADD]]"
  - "[[CMD]]"
  - "[[ENTRYPOINT]]"
  - "[[VOLUME]]"
  - "[[EXPOSE]]"
  - "[[docker build]]"
  - "[[docker compose]]"
  - "[[compose.yaml]]"
_organized: true
---
# Dockerfile、Docker Compose 与 Kubernetes 入门关系

Dockerfile 解决“镜像如何构建”，Compose 解决“多个容器如何一起启动”，Kubernetes 解决“多主机、多副本、自动调度和自愈如何管理”。这三者是递进关系，不是互相替代关系。

## 关系整理

| 工具             | 管理对象    | 典型文件                                  |
| -------------- | ------- | ------------------------------------- |
| Dockerfile     | 单个镜像    | `Dockerfile`                          |
| docker build   | 镜像构建过程  | 构建上下文目录                               |
| Docker Compose | 单机多容器项目 | `compose.yaml` / `docker-compose.yml` |
| Kubernetes     | 集群级容器编排 | YAML 资源清单                             |

## Dockerfile

Dockerfile 是镜像构建描述文件，文件名通常就是 `Dockerfile`。指令建议大写，每条指令描述一层镜像如何构建。

构建命令：

```bash
docker build -t nginx:01 .
```

这里最后的 `.` 不是简单表示 Dockerfile 所在目录，而是构建上下文路径。Docker 会把上下文发送给 Docker daemon，Dockerfile 中的 `COPY`、`ADD` 只能引用上下文里的文件。

建议配 `.dockerignore`，避免把日志、压缩包、`.git`、临时文件传进构建上下文。

```dockerignore
.git
*.log
*.tar.gz
node_modules
.env
```

## 常用指令

| 指令           | 作用                       |
| ------------ | ------------------------ |
| `FROM`       | 指定基础镜像，必须是第一条有效指令        |
| `RUN`        | 构建镜像时执行命令                |
| `COPY`       | 从构建上下文复制文件到镜像            |
| `ADD`        | 复制文件，支持 URL 和自动解压 tar，慎用 |
| `WORKDIR`    | 设置工作目录，不存在会自动创建          |
| `ENV`        | 设置环境变量                   |
| `EXPOSE`     | 声明容器服务端口，不等于自动发布端口       |
| `VOLUME`     | 定义匿名卷挂载点                 |
| `CMD`        | 容器默认启动命令或默认参数，可被覆盖       |
| `ENTRYPOINT` | 容器入口命令，常与 CMD 配合         |

## FROM

```dockerfile
FROM centos:7
FROM nginx:1.25
```

`FROM` 指定基础镜像。生产环境应尽量固定 tag，不要长期依赖 `latest`，否则重建镜像时可能得到不同结果。

## RUN

```dockerfile
RUN yum install -y wget
RUN ["yum", "install", "-y", "vim"]
```

更常见的优化写法是把安装、清理缓存放在同一层：

```dockerfile
RUN yum install -y make gcc pcre-devel zlib-devel     && yum clean all     && rm -rf /var/cache/yum
```

## EXPOSE

```dockerfile
EXPOSE 80 8080
```

`EXPOSE` 只是声明端口。真正映射到宿主机，需要运行容器时指定：

```bash
docker run -p 8080:80 image
```

## WORKDIR

```dockerfile
WORKDIR /app
```

相对路径会基于前一个 `WORKDIR`：

```dockerfile
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd
# /a/b/c
```

## ENV

```dockerfile
ENV NGX_VER=1.20.0
RUN echo $NGX_VER
```

`ENV` 设置的变量在后续构建层和容器运行时都可见。敏感信息不要写入镜像 ENV，避免被 `docker inspect` 看到。

## ADD 与 COPY

```dockerfile
COPY index.html /usr/share/nginx/html/
ADD nginx-1.20.1.tar.gz /app/
```

一般优先使用 `COPY`。`ADD` 的自动解压和 URL 下载能力容易导致构建行为不透明。需要下载文件时，通常用 `RUN curl -fSL ...` 更可控。

## VOLUME

```dockerfile
VOLUME /data
```

它会让容器运行时自动创建匿名卷，避免数据写入容器层。实际生产更推荐运行时显式指定命名卷：

```bash
docker run -v mydata:/data image
```

## CMD 与 ENTRYPOINT

`CMD` 是默认命令或默认参数，容易被 `docker run image xxx` 覆盖。`ENTRYPOINT` 是入口命令，`CMD` 常作为它的默认参数。

```dockerfile
ENTRYPOINT ["/bin/echo"]
CMD ["hello,world"]
```

运行：

```bash
docker run abc1000
# hello,world

docker run abc1000 welcome
# welcome
```

如果容器启动参数需要灵活追加，常用 `ENTRYPOINT + CMD`；如果只是定义默认启动命令，直接用 `CMD` 即可。

## Dockerfile 构建 Nginx 示例

```dockerfile
FROM centos:7

RUN curl -s -o /etc/yum.repos.d/CentOS-Base.repo     http://mirrors.aliyun.com/repo/Centos-7.repo

WORKDIR /app
ADD http://nginx.org/download/nginx-1.20.1.tar.gz ./

RUN tar xf nginx-1.20.1.tar.gz     && yum install -y make gcc pcre pcre-devel zlib zlib-devel vim     && cd nginx-1.20.1     && ./configure --prefix=/nginx     && make     && make install     && yum clean all

EXPOSE 80
CMD ["/nginx/sbin/nginx", "-g", "daemon off;"]
```

`nginx -g "daemon off;"` 的作用是让 Nginx 保持前台运行。容器需要一个前台主进程，不能把服务启动到后台后直接结束脚本。

## Docker Compose

Compose 用一个 YAML 文件定义多个服务、网络和数据卷。旧命令是 `docker-compose`，现代 Docker 更推荐 Compose v2：

```bash
docker compose version
docker compose up -d
```

默认文件名可以是：

```text
compose.yaml
compose.yml
docker-compose.yaml
docker-compose.yml
```

## YAML 规则

| 规则           | 说明                  |
| ------------ | ------------------- |
| `key: value` | 冒号后必须有空格            |
| 缩进           | 只能用空格，不能用 tab       |
| 大小写          | 敏感                  |
| 列表           | 用 `-` 表示            |
| 字符串          | 一般可以不加引号，但端口映射建议加引号 |

## Compose 基础示例

```yaml
services:
  web:
    image: nginx
    ports:
      - "88:80"
```

如果使用较老的 `docker-compose` 1.x，通常需要 `version: '3'`；Compose v2 规范中 `version` 已不是必须字段，但保留通常也能兼容旧习惯。

启动：

```bash
docker compose up -d
```

停止并删除容器和网络：

```bash
docker compose down
```

## Compose 常用字段

| 字段               | 作用                  |
| ---------------- | ------------------- |
| `image`          | 指定镜像                |
| `build`          | 指定构建上下文和 Dockerfile |
| `container_name` | 指定容器名               |
| `command`        | 覆盖默认启动命令            |
| `environment`    | 设置环境变量              |
| `env_file`       | 从文件读取环境变量           |
| `ports`          | 端口映射                |
| `volumes`        | 数据卷挂载               |
| `networks`       | 网络配置                |
| `depends_on`     | 启动顺序依赖              |
| `healthcheck`    | 健康检查                |
| `restart`        | 重启策略                |
| `ulimits`        | 资源限制                |

### build

```yaml
services:
  webapp:
    build:
      context: /data
      dockerfile: Dockerfile
```

### depends_on

```yaml
services:
  web:
    build: .
    depends_on:
      - db
      - redis
```

`depends_on` 只能保证启动顺序，不保证依赖服务内部已经可用。数据库需要初始化时间时，应配合 healthcheck 或应用侧重试。

### healthcheck

```yaml
services:
  web:
    image: nginx
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### volumes

```yaml
services:
  mysql:
    image: mysql:5.7
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

## Compose 部署 WordPress

```yaml
services:
  wordpress:
    image: wordpress
    restart: always
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    volumes:
      - wordpress:/var/www/html

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_ROOT_PASSWORD: rootpass
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:
```

启动：

```bash
docker compose up -d
```

注意原始示例中的 `MYSQL_RANDOM_ROOT_PASSWORD: '123456'` 语义不准确。这个变量用于让 MySQL 随机生成 root 密码，不应该填固定密码；要固定 root 密码应使用 `MYSQL_ROOT_PASSWORD`。

## Compose 命令

```bash
docker compose config
docker compose up -d
docker compose up --build -d
docker compose down
docker compose ps
docker compose logs -f
docker compose exec web bash
docker compose restart
docker compose stop
docker compose start
docker compose images
docker compose top
docker compose version
```

旧版命令格式：

```bash
docker-compose up -d
```

新环境优先使用：

```bash
docker compose up -d
```

## Compose 完整结构示例

```yaml
services:
  web_go_app:
    build:
      context: /data
      dockerfile: Dockerfile
    ports:
      - "8081:8080"
    volumes:
      - web_go_app:/app
    container_name: go_app
    restart: always
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000
    depends_on:
      - db_mysql
      - db_redis
      - web_ngx
    networks:
      - web

  web_ngx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - web_ngx_html:/usr/share/nginx/html
      - web_ngx_conf:/etc/nginx/conf.d
    container_name: ngx
    restart: always
    networks:
      - web

  db_mysql:
    image: mysql:5.7
    ports:
      - "3306:3306"
    env_file:
      - /data/.mysql_env
    volumes:
      - db_mysql_data:/var/lib/mysql
    restart: always
    container_name: mysql
    networks:
      - web

  db_redis:
    image: redis:5.0.0
    ports:
      - "6379:6379"
    restart: always
    container_name: redis
    volumes:
      - db_redis_data:/data
    networks:
      - web

networks:
  web:

volumes:
  web_go_app:
  web_ngx_html:
  web_ngx_conf:
  db_mysql_data:
  db_redis_data:
```

## Kubernetes 的位置

Compose 适合单机多容器项目，Kubernetes 适合集群级容器编排。K8s 关注的不只是启动容器，还包括调度、服务发现、滚动更新、自动恢复、水平扩容、配置和密钥管理。

| Compose    | Kubernetes                                   |
| ---------- | -------------------------------------------- |
| 单机项目编排     | 多节点集群编排                                      |
| `services` | `Deployment` / `StatefulSet` / `Service`     |
| `volumes`  | `PersistentVolume` / `PersistentVolumeClaim` |
| `networks` | CNI 网络和 Service 发现                           |
| `restart`  | 控制器自动拉起 Pod                                  |

学习顺序上，先把 Dockerfile 和 Compose 写熟，再看 K8s 的 Pod、Deployment、Service、ConfigMap、Secret，会更顺。

## 易错点

| 问题                                        | 处理                          |
| ----------------------------------------- | --------------------------- |
| 把 `docker build .` 的 `.` 当成 Dockerfile 路径 | 它是构建上下文路径                   |
| Dockerfile 泄露 `.env`                      | 使用 `.dockerignore`          |
| `EXPOSE` 后宿主机仍不能访问                        | 还需要 `-p` 或 Compose `ports`  |
| `CMD` 参数被覆盖                               | 需要理解 CMD 与 ENTRYPOINT 的组合   |
| Compose 缩进报错                              | YAML 只能用空格，不能用 tab          |
| `depends_on` 后应用仍连不上数据库                   | 它不等于健康检查，需要重试或 healthcheck  |
| MySQL root 密码变量写错                         | 固定密码用 `MYSQL_ROOT_PASSWORD` |
