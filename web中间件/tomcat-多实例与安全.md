---
type: Practice
status: Organized
Belongs to: "[[Tomcat]]"
Related to:
  - "[[tomcat-部署与配置]]"
Has:
  - "[[Tomcat 多实例]]"
source: day18_tomcat.md
created: 2026-06-14
updated: 2026-06-14
---

# Tomcat 多实例与安全

## 多实例

多实例复用服务器资源，核心要求是端口不能冲突。

复制实例：

```bash
cp -a /tools/apache-tomcat-9.0.62 /tools/tomcat2
```

修改 `server.xml`：

```xml
<Server port="8006" shutdown="SHUTDOWN">

<Connector port="8081" protocol="HTTP/1.1"
           connectionTimeout="20000"
           redirectPort="8444" />
```

验证：

```bash
ss -lntp | grep java
```

## 管理后台

`tomcat-users.xml` 配置用户与角色：

```xml
<role rolename="manager-gui"/>
<role rolename="admin-gui"/>
<user username="tomcat" password="tomcat"
      roles="manager-gui,admin-gui"/>
```

| 角色 | 作用 |
| --- | --- |
| `manager-gui` | HTML 管理界面 |
| `manager-script` | 文本接口 |
| `admin-gui` | Host Manager HTML |

默认 manager 只允许本机访问。放开限制需修改 `webapps/manager/META-INF/context.xml`，但不要在生产公网环境放开。

## 安全基线

| 项目 | 建议 |
| --- | --- |
| 默认 Web 应用 | 删除 `docs`、`examples`、`manager`、`host-manager` |
| 管理后台 | 非必要不开放公网 |
| 端口 | 8080 只对 Nginx 或内网开放 |
| 用户 | 使用普通用户运行 Tomcat |
| AJP | 非必要禁用 |
| Shutdown 端口 | 修改默认 shutdown 字符串 |

删除默认页面：

```bash
rm -rf /tools/apache-tomcat-9.0.62/webapps/*
```

## JVM 观测

```bash
jmap -heap <tomcat_pid>
jps -l
ss -lntp | grep java
```

## 易错点

| 问题 | 处理 |
| --- | --- |
| 多实例启动失败 | 检查 8005/8080/8009 是否冲突 |
| 管理后台打不开 | 检查角色、用户、context.xml 限制 |
| 默认页面暴露 | 删除不需要的默认 Web 应用 |
