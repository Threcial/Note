---
type: knowledge
status: evergreen
created: 2026-06-14
updated: 2026-06-14
related_to:
  - "[[dockerfile-参考]]"
  - "[[docker-命令行参考]]"
  - "[[容器编排]]"
has:
  - "[[docker compose]]"
  - "[[compose.yaml]]"
_organized: true
---

# Docker Compose 入门

Compose 解决"多个容器如何一起启动"。它用一个 YAML 文件定义多个服务、网络和数据卷。

旧命令是 `docker-compose`，现代 Docker 更推荐 Compose v2：

```bash
docker compose version
docker compose up -d
```

默认文件名（按优先级）：

```text
compose.yaml
compose.yml
docker-compose.yaml
docker-compose.yml
```

## YAML 规则

| 规则 | 说明 |
| --- | --- |
| `key: value` | 冒号后必须有空格 |
| 缩进 | 只能用空格，不能用 tab |
| 大小写 | 敏感 |
| 列表 | 用 `-` 表示 |
| 字符串 | 一般可以不加引号，但端口映射建议加引号 |

## 基础示例

```yaml
services:
  web:
    image: nginx
    ports:
      - "88:80"
```

如果使用较老的 `docker-compose` 1.x，通常需要 `version: '3'`；Compose v2 规范中 `version` 已不是必须字段。

## 常用字段

| 字段 | 作用 |
| --- | --- |
| `image` | 指定镜像 |
| `build` | 指定构建上下文和 Dockerfile |
| `container_name` | 指定容器名 |
| `command` | 覆盖默认启动命令 |
| `environment` | 设置环境变量 |
| `env_file` | 从文件读取环境变量 |
| `ports` | 端口映射 |
| `volumes` | 数据卷挂载 |
| `networks` | 网络配置 |
| `depends_on` | 启动顺序依赖 |
| `healthcheck` | 健康检查 |
| `restart` | 重启策略 |
| `ulimits` | 资源限制 |

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

## WordPress 部署

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

注意 `MYSQL_RANDOM_ROOT_PASSWORD` 用于让 MySQL 随机生成 root 密码，不应填固定值。固定 root 密码应使用 `MYSQL_ROOT_PASSWORD`。

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
```

新环境优先使用 `docker compose`（空格）而非旧版 `docker-compose`。

## 完整结构示例

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

## 易错点

| 问题 | 处理 |
| --- | --- |
| Compose 缩进报错 | YAML 只能用空格，不能用 tab |
| `depends_on` 后应用仍连不上数据库 | 它不等于健康检查，需要重试或 healthcheck |
| MySQL root 密码变量写错 | 固定密码用 `MYSQL_ROOT_PASSWORD` |
