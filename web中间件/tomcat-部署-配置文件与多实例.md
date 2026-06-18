---
type: Practice
status: Organized
source: day18_tomcat.md
created: 2026-06-14
updated: 2026-06-14
Belongs to: "[[Tomcat]]"
Related to:
  - "[[Nginx 反向代理]]"
  - "[[JDK]]"
  - "[[systemd 服务管理]]"
Has:
  - "[[server.xml]]"
  - "[[Tomcat 多实例]]"
  - "[[Tomcat 端口]]"
---
# Tomcat 部署、配置文件与多实例

Tomcat 是 Java Web 容器，适合运行 JSP、Servlet 和 Java Web 应用。生产中通常不建议 Tomcat 直接暴露给公网，而是由 Nginx 或 Apache 处理静态资源、TLS、限速、日志和反向代理，再把动态请求转发给 Tomcat。

Tomcat 部署时需要同时关注 JDK、端口、内存参数、日志、默认应用、管理后台权限和多实例端口冲突。

## 关系总览

| 条目                 | 关系                 |
| ------------------ | ------------------ |
| Tomcat             | Java Web 容器        |
| JDK                | Tomcat 运行依赖        |
| `catalina.sh`      | Tomcat 启动主脚本       |
| `server.xml`       | 端口、连接器、虚拟主机等核心配置   |
| `web.xml`          | 默认欢迎页、会话等 Web 应用配置 |
| `tomcat-users.xml` | 管理后台用户与角色          |
| Nginx              | 常作为 Tomcat 前置反向代理  |

## Tomcat 与 Apache、Nginx 的关系

| 软件           | 主要定位                     |
| ------------ | ------------------------ |
| Apache HTTPD | 老牌 Web 服务器，模块丰富          |
| Nginx        | 轻量 Web 服务器、反向代理、负载均衡     |
| Tomcat       | Java Web 容器，处理动态 Java 请求 |

常见架构：

```text
Client -> Nginx -> Tomcat
```

Nginx 负责公网入口、静态资源、HTTPS、日志、限速和反向代理；Tomcat 专注 Java 应用运行。

## JDK 与 Tomcat 安装

假设安装包已经上传到 `/tools`：

```bash
cd /tools
tar zxvf jdk-8u333-linux-x64.tar.gz
tar zxvf apache-tomcat-9.0.62.tar.gz
```

配置 JDK 环境变量：

```bash
vim /etc/profile
```

```bash
export JAVA_HOME=/tools/jdk1.8.0_333
export PATH=$JAVA_HOME/bin:$PATH
```

让环境变量生效：

```bash
source /etc/profile
java -version
```

也可以在 Tomcat 自身脚本中指定 `JAVA_HOME`，适合多 JDK 并存场景。

## Tomcat 内存参数

在 `catalina.sh` 中添加：

```bash
JAVA_OPTS="$JAVA_OPTS -Xms1024m -Xmx1024m -Xss1024K"
```

常见参数：

| 参数     | 含义        |
| ------ | --------- |
| `-Xms` | JVM 初始堆内存 |
| `-Xmx` | JVM 最大堆内存 |
| `-Xss` | 单线程栈大小    |

如果要改成 2G：

```bash
JAVA_OPTS="$JAVA_OPTS -Xms2048m -Xmx2048m -Xss1024K"
```

内存参数不能只看机器总内存，还要考虑系统、Nginx、日志、监控 agent、多个 Tomcat 实例和其他服务的占用。

## 启动与验证

启动：

```bash
cd /tools/apache-tomcat-9.0.62/bin
./catalina.sh start
```

停止：

```bash
./catalina.sh stop
```

如果无法正常停止，再查进程，不要直接养成 `kill -9` 的习惯：

```bash
ps -ef | grep tomcat
ss -lntp | grep java
```

日志：

```bash
tail -f /tools/apache-tomcat-9.0.62/logs/catalina.out
```

浏览器验证：

```yaml
http://服务器IP:8080/
```

命令行验证：

```bash
curl -I http://127.0.0.1:8080/
```

## Tomcat 常见端口

| 端口   | 作用                 |
| ---- | ------------------ |
| 8080 | HTTP Web 访问端口      |
| 8005 | Shutdown 关闭端口      |
| 8009 | AJP 端口，现代环境通常不建议开放 |

`8005` 是关闭端口，生产中应修改默认 shutdown 字符串，甚至关闭该端口，避免被误用。

```xml
<Server port="8005" shutdown="SHUTDOWN">
```

HTTP 连接器：

```xml
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

AJP 连接器历史上出现过安全问题，非必要不要启用；如果启用，应限制监听地址和访问来源。

## Tomcat 目录结构

| 目录        | 作用         |
| --------- | ---------- |
| `bin`     | 启动、关闭、功能脚本 |
| `conf`    | 配置文件       |
| `lib`     | Tomcat 依赖库 |
| `logs`    | 日志目录       |
| `webapps` | Web 应用发布目录 |
| `temp`    | 临时文件       |
| `work`    | JSP 编译后的文件 |

关键文件：

| 文件                      | 作用          |
| ----------------------- | ----------- |
| `bin/startup.sh`        | 启动脚本        |
| `bin/shutdown.sh`       | 关闭脚本        |
| `bin/catalina.sh`       | 核心控制脚本      |
| `conf/server.xml`       | 端口、连接器、Host |
| `conf/web.xml`          | 默认 Web 配置   |
| `conf/tomcat-users.xml` | 管理用户配置      |

日志文件：

| 文件                                    | 作用                |
| ------------------------------------- | ----------------- |
| `catalina.out`                        | 控制台输出汇总           |
| `catalina.YYYY-MM-DD.log`             | Catalina 日志       |
| `localhost_access_log.YYYY-MM-DD.txt` | 访问日志              |
| `manager.*.log`                       | manager 应用日志      |
| `host-manager.*.log`                  | host-manager 应用日志 |

## server.xml 核心配置

### Shutdown 端口

```xml
<Server port="8005" shutdown="SHUTDOWN">
```

生产建议修改 shutdown 字符串：

```xml
<Server port="8005" shutdown="复杂随机字符串">
```

或者根据版本和场景禁用关闭端口。

### Web 端口

```xml
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443" />
```

### 连接数和线程数

```xml
<Connector port="8080" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8443"
           maxConnections="1000"
           maxThreads="100"
           acceptCount="1000" />
```

| 参数               | 含义             |
| ---------------- | -------------- |
| `maxConnections` | 最大连接数          |
| `maxThreads`     | 最大工作线程数        |
| `acceptCount`    | 线程和连接忙时的等待队列长度 |

如果 `maxConnections=1000`、`acceptCount=1000`，达到连接和排队上限后，新的连接可能被拒绝或超时。

## web.xml

欢迎页：

```xml
<welcome-file-list>
    <welcome-file>index.html</welcome-file>
    <welcome-file>index.htm</welcome-file>
    <welcome-file>index.jsp</welcome-file>
</welcome-file-list>
```

会话超时时间：

```xml
<session-config>
    <session-timeout>30</session-timeout>
</session-config>
```

单位通常是分钟。

## tomcat-users.xml 与管理后台

示例：

```xml
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<role rolename="manager-jmx"/>
<role rolename="manager-status"/>
<role rolename="admin-gui"/>
<role rolename="admin-script"/>

<user username="tomcat" password="tomcat"
      roles="manager-gui,manager-script,manager-jmx,manager-status,admin-gui,admin-script"/>
```

角色：

| 角色               | 作用                |
| ---------------- | ----------------- |
| `manager-gui`    | HTML 管理界面         |
| `manager-script` | 文本接口              |
| `manager-jmx`    | JMX 代理            |
| `manager-status` | 状态查看              |
| `admin-gui`      | Host Manager HTML |
| `admin-script`   | Host Manager 文本接口 |

默认 manager 和 host-manager 通常只允许本机访问。如果要放开，需要修改：

```text
webapps/manager/META-INF/context.xml
webapps/host-manager/META-INF/context.xml
```

把 IP 限制改为允许范围。不要直接在公网环境放开 `allow="^.*$"`，这会暴露管理后台。

## 安全基线

| 项目        | 建议                                                |
| --------- | ------------------------------------------------- |
| 默认 Web 应用 | 删除不需要的 `docs`、`examples`、`host-manager`、`manager` |
| 管理后台      | 非必要不开放公网                                          |
| 端口        | Tomcat 8080 只对 Nginx 或内网开放                        |
| 用户        | 使用普通用户运行 Tomcat                                   |
| 日志        | 配置切割和保留周期                                         |
| JDK       | 不使用过旧版本                                           |
| AJP       | 非必要禁用                                             |

删除默认页面：

```bash
rm -rf /tools/apache-tomcat-9.0.62/webapps/*
```

部署业务时再放入自己的应用包。

## Tomcat 多实例

多实例的意义是复用服务器资源，在同一台机器上运行多个 Tomcat。核心要求是端口不能冲突。

复制实例：

```bash
cp -a /tools/apache-tomcat-9.0.62 /tools/tomcat2
```

修改 `tomcat2/conf/server.xml`：

```xml
<Server port="8006" shutdown="SHUTDOWN">
```

```xml
<Connector port="8081" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8444" />
```

如果启用 AJP，也要修改：

```xml
<Connector protocol="AJP/1.3"
           address="127.0.0.1"
           port="8010"
           redirectPort="8444" />
```

验证：

```bash
ss -lntp | grep java
```

预期能看到多个 Java 进程监听不同端口。

## Tomcat 与 Nginx 配合

Nginx 反向代理 Tomcat：

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
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

如果应用需要知道真实协议和来源 IP，应让 Java 应用读取代理头，或在 Tomcat 中配置 RemoteIpValve。

## JVM 观测

查看堆内存：

```bash
jmap -heap <tomcat_pid>
```

查看 Java 进程：

```bash
jps -l
ps -ef | grep java
```

查看端口：

```bash
ss -lntp | grep java
```

## systemd 管理补充

长期运行建议使用 systemd 管理 Tomcat。

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
Group=root
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

生产中更建议创建专用用户运行 Tomcat，而不是 root。

```bash
systemctl daemon-reload
systemctl enable --now tomcat
systemctl status tomcat
```

## 命令速查

| 目的         | 命令                               |
| ---------- | -------------------------------- |
| 查看 Java 版本 | `java -version`                  |
| 启动 Tomcat  | `./catalina.sh start`            |
| 停止 Tomcat  | `./catalina.sh stop`             |
| 查看日志       | `tail -f logs/catalina.out`      |
| 查看端口       | `ss -lntp                        |
| 查看进程       | `ps -ef                          |
| 查看堆        | `jmap -heap <pid>`               |
| 测试访问       | `curl -I http://127.0.0.1:8080/` |

## 易错点

| 问题          | 处理                                   |
| ----------- | ------------------------------------ |
| Tomcat 启动失败 | 检查 `JAVA_HOME`、端口冲突、日志               |
| 多实例启动失败     | 检查 8005/8080/8009 是否冲突               |
| 关闭失败        | 先看 `shutdown.sh` 日志，不要直接依赖 `kill -9` |
| 管理后台打不开     | 检查角色、用户、context.xml 访问限制             |
| 8080 暴露公网   | 建议只允许内网或 Nginx 访问                    |
| 默认页面暴露      | 删除不需要的默认 Web 应用                      |

## 关联条目

| 类型 | 条目                                        |
| -- | ----------------------------------------- |
| 前置 | [[JDK]]、[[Linux 端口]]、[[systemd 服务管理]]     |
| 核心 | [[Tomcat]]、[[server.xml]]、[[Tomcat 多实例]]  |
| 延伸 | [[Nginx 反向代理]]、[[Java Web 部署]]、[[JVM 监控]] |
