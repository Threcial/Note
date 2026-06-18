---
type: Practice
status: Organized
Belongs to: "[[Tomcat]]"
Related to:
  - "[[JDK]]"
  - "[[nginx-反向代理与限速]]"
Has:
  - "[[server.xml]]"
source: day18_tomcat.md
created: 2026-06-14
updated: 2026-06-14
---

# Tomcat 部署与配置

Tomcat 是 Java Web 容器。生产中不建议 Tomcat 直接暴露给公网，应由 Nginx 处理静态资源和反向代理。

## JDK 安装

```bash
tar zxvf jdk-8u333-linux-x64.tar.gz -C /tools
```

配置环境变量：

```bash
export JAVA_HOME=/tools/jdk1.8.0_333
export PATH=$JAVA_HOME/bin:$PATH
```

## Tomcat 安装

```bash
tar zxvf apache-tomcat-9.0.62.tar.gz -C /tools
```

### 内存参数

在 `catalina.sh` 中添加：

```bash
JAVA_OPTS="$JAVA_OPTS -Xms1024m -Xmx1024m -Xss1024K"
```

| 参数 | 含义 |
| --- | --- |
| `-Xms` | JVM 初始堆内存 |
| `-Xmx` | JVM 最大堆内存 |
| `-Xss` | 单线程栈大小 |

## 目录结构

| 目录 | 作用 |
| --- | --- |
| `bin` | 启动、关闭脚本 |
| `conf` | 配置文件 |
| `lib` | 依赖库 |
| `logs` | 日志目录 |
| `webapps` | 应用发布目录 |
| `work` | JSP 编译后的文件 |

关键文件：`bin/startup.sh`、`bin/shutdown.sh`、`bin/catalina.sh`、`conf/server.xml`。

日志文件：`catalina.out`、`catalina.YYYY-MM-DD.log`、`localhost_access_log.YYYY-MM-DD.txt`。

## 端口

| 端口 | 作用 |
| --- | --- |
| 8080 | HTTP 访问端口 |
| 8005 | Shutdown 关闭端口 |
| 8009 | AJP 端口，现代环境不建议开放 |

## server.xml 核心配置

```xml
<Server port="8005" shutdown="SHUTDOWN">

<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443"
           maxConnections="1000"
           maxThreads="100"
           acceptCount="1000" />
```

| 参数 | 含义 |
| --- | --- |
| `maxConnections` | 最大连接数 |
| `maxThreads` | 最大工作线程数 |
| `acceptCount` | 等待队列长度 |

## 启动与验证

```bash
cd /tools/apache-tomcat-9.0.62/bin
./catalina.sh start
./catalina.sh stop
```

```bash
tail -f logs/catalina.out
curl -I http://127.0.0.1:8080/
```

## systemd 管理

```ini
[Unit]
Description=Apache Tomcat
After=network.target

[Service]
Type=forking
Environment=JAVA_HOME=/tools/jdk1.8.0_333
Environment=CATALINA_HOME=/tools/apache-tomcat-9.0.62
ExecStart=/tools/apache-tomcat-9.0.62/bin/startup.sh
ExecStop=/tools/apache-tomcat-9.0.62/bin/shutdown.sh
User=root
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## Nginx 反向代理 Tomcat

```nginx
upstream tomcat {
    server 127.0.0.1:8080;
}

server {
    listen 80;
    server_name java.example.com;

    location / {
        proxy_pass http://tomcat;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## 易错点

| 问题 | 处理 |
| --- | --- |
| Tomcat 启动失败 | 检查 `JAVA_HOME`、端口冲突、日志 |
| 8080 暴露公网 | 建议只允许内网或 Nginx 访问 |
| 关闭失败 | 先用 `shutdown.sh`，不要直接 `kill -9` |
