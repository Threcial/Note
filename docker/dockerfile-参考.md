---
type: Reference
status: Organized
Belongs to: "[[Docker]]"
Related to:
  - "[[docker-基础概念]]"
  - "[[docker-命令行参考]]"
Has:
  - "[[FROM]]"
  - "[[RUN]]"
  - "[[COPY]]"
  - "[[ADD]]"
  - "[[CMD]]"
  - "[[ENTRYPOINT]]"
  - "[[docker build]]"
source: ""
created: 2026-06-14
updated: 2026-06-14
---

# Dockerfile 参考

Dockerfile 解决"镜像如何构建"。文件名通常就是 `Dockerfile`，指令建议大写，每条指令描述一层镜像如何构建。

## docker build

```bash
docker build -t nginx:01 .
```

这里的 `.` 不是简单表示 Dockerfile 所在目录，而是 **构建上下文路径**。Docker 会把上下文发送给 Docker daemon，Dockerfile 中的 `COPY`、`ADD` 只能引用上下文里的文件。

建议配 `.dockerignore`，避免把日志、压缩包、`.git`、临时文件传进构建上下文：

```dockerignore
.git
*.log
*.tar.gz
node_modules
.env
```

## 常用指令

| 指令 | 作用 |
| --- | --- |
| `FROM` | 指定基础镜像，必须是第一条有效指令 |
| `RUN` | 构建镜像时执行命令 |
| `COPY` | 从构建上下文复制文件到镜像 |
| `ADD` | 复制文件，支持 URL 和自动解压 tar，慎用 |
| `WORKDIR` | 设置工作目录，不存在会自动创建 |
| `ENV` | 设置环境变量 |
| `EXPOSE` | 声明容器服务端口，不等于自动发布端口 |
| `VOLUME` | 定义匿名卷挂载点 |
| `CMD` | 容器默认启动命令或默认参数，可被覆盖 |
| `ENTRYPOINT` | 容器入口命令，常与 CMD 配合 |

## FROM

```dockerfile
FROM centos:7
FROM nginx:1.25
```

生产环境应尽量固定 tag，不要长期依赖 `latest`，否则重建镜像时可能得到不同结果。

## RUN

```dockerfile
RUN yum install -y wget
RUN ["yum", "install", "-y", "vim"]
```

更常见的优化写法是把安装、清理缓存放在同一层：

```dockerfile
RUN yum install -y make gcc pcre-devel zlib-devel \
    && yum clean all \
    && rm -rf /var/cache/yum
```

## EXPOSE

```dockerfile
EXPOSE 80 8080
```

`EXPOSE` 只是声明端口。真正映射到宿主机需要运行容器时指定 `docker run -p 8080:80 image`。

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

```bash
docker run abc1000
# hello,world

docker run abc1000 welcome
# welcome
```

如果容器启动参数需要灵活追加，常用 `ENTRYPOINT + CMD`；如果只是定义默认启动命令，直接用 `CMD` 即可。

## Nginx 构建示例

```dockerfile
FROM centos:7

RUN curl -s -o /etc/yum.repos.d/CentOS-Base.repo \
    http://mirrors.aliyun.com/repo/Centos-7.repo

WORKDIR /app
ADD http://nginx.org/download/nginx-1.20.1.tar.gz ./

RUN tar xf nginx-1.20.1.tar.gz \
    && yum install -y make gcc pcre pcre-devel zlib zlib-devel vim \
    && cd nginx-1.20.1 \
    && ./configure --prefix=/nginx \
    && make \
    && make install \
    && yum clean all

EXPOSE 80
CMD ["/nginx/sbin/nginx", "-g", "daemon off;"]
```

`nginx -g "daemon off;"` 的作用是让 Nginx 保持前台运行。容器需要一个前台主进程，不能把服务启动到后台后直接结束脚本。

## 易错点

| 问题 | 处理 |
| --- | --- |
| 把 `docker build .` 的 `.` 当成 Dockerfile 路径 | 它是构建上下文路径 |
| Dockerfile 泄露 `.env` | 使用 `.dockerignore` |
| `EXPOSE` 后宿主机仍不能访问 | 还需要 `-p` 或 Compose `ports` |
| `CMD` 参数被覆盖 | 需要理解 CMD 与 ENTRYPOINT 的组合 |
