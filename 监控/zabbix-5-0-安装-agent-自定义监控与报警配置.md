---
type: Practice
status: Organized
source: day25.md
created: 2026-06-14
updated: 2026-06-14
Belongs to: "[[Zabbix]]"
Related to:
  - "[[Nginx]]"
  - "[[MySQL]]"
Has:
  - "[[zabbix-server]]"
  - "[[zabbix-agent]]"
  - "[[UserParameter]]"
  - "[[Zabbix 自动注册]]"
  - "[[Zabbix 触发器]]"
  - "[[Zabbix 报警媒介]]"
---
# Zabbix 5.0 安装、Agent、自定义监控与报警配置

Zabbix 的链路可以拆成四层：server 负责收集和处理数据，agent 负责在被监控主机上采集数据，item/trigger/action 定义“采什么、什么算异常、异常后做什么”，media 定义“用什么方式通知”。

## 关系整理

| 对象              | 作用                   |
| --------------- | -------------------- |
| 主机              | 被监控对象                |
| 监控项 item        | 采集一个具体指标             |
| 触发器 trigger     | 根据监控项判断是否异常          |
| 动作 action       | 异常发生后执行通知或命令         |
| 媒介类型 media type | 邮件、Webhook、企业微信等发送方式 |
| 用户报警媒介          | 某个用户实际绑定的接收渠道        |
| 模板 template     | 监控项、触发器、图形的复用集合      |

## 初始化系统设置

```bash
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config
```

实验环境可以关闭防火墙和 SELinux；生产环境更推荐精确放行端口，例如 10051、10050、80/443，并保留安全策略。

## 安装 Zabbix Server

添加 Zabbix 5.0 源并替换为镜像地址：

```bash
rpm -Uvh https://mirrors.aliyun.com/zabbix/zabbix/5.0/rhel/7/x86_64/zabbix-release-5.0-1.el7.noarch.rpm
sed -i 's#http://repo.zabbix.com#https://mirrors.aliyun.com/zabbix#' /etc/yum.repos.d/zabbix.repo
yum clean all
yum makecache
```

安装 server 和 agent：

```bash
yum install zabbix-server-mysql zabbix-agent -y
```

安装 SCL，以便使用较高版本 PHP：

```bash
yum install centos-release-scl -y
```

启用前端源：

```ini
[zabbix-frontend]
enabled=1
```

CentOS 7 的 SCLo 源经常失效，需要把 `CentOS-SCLo-scl.repo` 和 `CentOS-SCLo-scl-rh.repo` 改成可用镜像源。

安装前端：

```bash
yum install zabbix-web-mysql-scl zabbix-apache-conf-scl -y
```

## 数据库初始化

安装并启动 MariaDB：

```bash
yum install mariadb-server -y
systemctl start mariadb.service
mysql_secure_installation
```

创建库和用户：

```sql
CREATE DATABASE zabbix CHARACTER SET utf8 COLLATE utf8_bin;
GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost IDENTIFIED BY 'zabbix';
FLUSH PRIVILEGES;
```

导入初始化 SQL：

```bash
cd /usr/share/doc/zabbix-server-mysql-5.0.47
gunzip create.sql.gz
mysql -uroot -p zabbix < create.sql
```

如果版本目录不一致，先查：

```bash
rpm -ql zabbix-server-mysql | grep create.sql
```

## zabbix_server.conf

```ini
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=zabbix
DBSocket=/var/lib/mysql/mysql.sock
StartPollers=100
Timeout=30
```

`StartPollers` 不是越大越好。监控主机越多、被动采集越多，需要更多 poller；但进程过多也会增加 server 自身负载。

启动服务：

```bash
systemctl start zabbix-server.service
```

## PHP 时区与 Web 服务

修改：

```bash
vim /etc/opt/rh/rh-php72/php-fpm.d/zabbix.conf
```

设置：

```ini
php_value[date.timezone] = Asia/Shanghai
```

启动：

```bash
systemctl start httpd.service
systemctl start rh-php72-php-fpm.service
systemctl enable httpd.service rh-php72-php-fpm.service zabbix-server.service mariadb.service
```

浏览器访问：

```yaml
http://服务器IP/zabbix
```

默认账号密码：

```text
Admin / zabbix
```

## 中文乱码

图形中文乱码时，把中文字体替换为 Zabbix 使用的字体文件：

```bash
cd /usr/share/zabbix/assets/fonts
cp graphfont.ttf graphfont.ttf.bak
# 上传 simhei.ttf 后
mv simhei.ttf graphfont.ttf
```

不要在交付文件里传播字体文件；只记录操作位置即可。

## 添加被监控主机

在被监控主机安装 agent：

```bash
rpm -ivh https://mirrors.aliyun.com/zabbix/zabbix/5.0/rhel/7/x86_64/zabbix-release-5.0-1.el7.noarch.rpm
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX
yum install -y zabbix-agent zabbix-sender
```

配置 `/etc/zabbix/zabbix_agentd.conf`：

```ini
Server=192.168.31.116
ServerActive=192.168.31.116
Hostname=nginx01
```

启动：

```bash
systemctl start zabbix-agent.service
systemctl enable zabbix-agent.service
```

`Server` 用于被动模式，server 主动连 agent 的 10050 端口取数据；`ServerActive` 用于主动模式，agent 主动连 server 的 10051 端口上报数据。

Zabbix Agent2 是新一代 agent，插件能力更强，监控 MySQL、Docker、Redis 等服务时更方便。已有 Zabbix 5.0 环境仍可继续使用 agentd，但后续可以考虑逐步用 agent2。

## zabbix_get 测试

server 上安装：

```bash
yum install -y zabbix-get
```

测试 agent：

```bash
zabbix_get -s 192.168.31.20 -k system.hostname
```

自定义监控项上线前，必须用 `zabbix_get` 在 server 侧验证，避免网页里配置成功但采集无数据。

## 自定义监控项 UserParameter

agent 配置包含目录：

```ini
Include=/etc/zabbix/zabbix_agentd.d/*.conf
```

创建配置文件：

```bash
vim /etc/zabbix/zabbix_agentd.d/custom.conf
```

示例：

```ini
UserParameter=connect,ss -ant | grep ESTAB | wc -l
UserParameter=test,echo 0
UserParameter=ccee,sh /tmp/1.sh
```

复杂逻辑不要直接堆在 `UserParameter` 后面，应该写成脚本，再由 UserParameter 调用。

修改后重启 agent：

```bash
systemctl restart zabbix-agent.service
```

server 测试：

```bash
zabbix_get -s 192.168.31.20 -k 'connect'
```

## 自动发现与自动注册

自动发现是 server 扫描网段查找 agent，资源消耗较大，通常不作为主方案。

自动注册是 agent 主动向 server 发送自己的信息，server 根据 `HostMetadata` 匹配动作并自动链接模板。

agent 配置：

```ini
Server=192.168.31.116
ServerActive=192.168.31.116
Hostname=jenkins001
HostMetadata=jenkins
```

Web 配置路径：

```text
配置 -> 动作 -> 自动注册动作 -> 创建动作
```

条件可以写：主机元数据包含 `jenkins`。操作选择链接到对应模板。

## 日志位置

```bash
tail -f /var/log/zabbix/zabbix_server.log
tail -f /var/log/zabbix/zabbix_agentd.log
```

如果 agent 主动模式失败，优先看 agent 日志；如果 server 侧创建主机失败，看 server 日志。

## Nginx 并发监控

开启 Nginx `stub_status`：

```nginx
server {
    location /nginx-status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
    }
}
```

获取状态：

```bash
curl http://127.0.0.1/nginx-status
```

字段含义：

| 字段       | 说明                      |
| -------- | ----------------------- |
| Active   | 当前活跃连接数                 |
| accepts  | 启动至今接收的连接数              |
| handled  | 成功处理的连接数                |
| requests | 总请求数                    |
| Reading  | 正在读取请求头的连接数             |
| Writing  | 正在返回响应的连接数              |
| Waiting  | keep-alive 下等待下一次请求的连接数 |

脚本示例：

```bash
#!/bin/bash
host="127.0.0.1"
port="80"

function ping {
  /sbin/pidof nginx | wc -w
}

function active {
  /usr/bin/curl -s "http://$host:$port/nginx-status" | awk '/Active/ {print $NF}'
}

function reading {
  /usr/bin/curl -s "http://$host:$port/nginx-status" | awk '/Reading/ {print $2}'
}

function writing {
  /usr/bin/curl -s "http://$host:$port/nginx-status" | awk '/Writing/ {print $4}'
}

function waiting {
  /usr/bin/curl -s "http://$host:$port/nginx-status" | awk '/Waiting/ {print $6}'
}

function accepts {
  /usr/bin/curl -s "http://$host:$port/nginx-status" | awk 'NR==3 {print $1}'
}

function handled {
  /usr/bin/curl -s "http://$host:$port/nginx-status" | awk 'NR==3 {print $2}'
}

function requests {
  /usr/bin/curl -s "http://$host:$port/nginx-status" | awk 'NR==3 {print $3}'
}

$1
```

UserParameter：

```ini
UserParameter=nginx.status[*],sh /etc/zabbix/zabbix_agentd.d/nginx_status.sh $1
```

测试：

```bash
zabbix_get -s 192.168.31.112 -k 'nginx.status[ping]'
```

## Web 场景监控

Web 场景适合模拟访问首页、登录页、接口健康检查。

配置路径：

```text
配置 -> 主机 -> Web 检测 -> 创建 Web 场景
```

步骤里配置 URL 和期望状态码，例如：

```yaml
URL: http://192.168.0.132/index.html
要求的状态码: 200
```

触发器可以基于响应码不等于 200：

```text
Response code for step "首页" of scenario "首页" <> 200
```

## 报警链路

报警不是只配一个触发器就结束。完整链路是：

```text
监控项 -> 触发器 -> 动作 -> 媒介类型 -> 用户报警媒介 -> 接收端
```

如果没有收到报警，按这条链路逐个排查：触发器是否进入问题状态，动作条件是否匹配，用户是否有对应媒介，时间段是否允许发送，媒介类型脚本或 webhook 是否成功。

## 企业微信机器人报警

企业微信机器人可以通过 Webhook 媒介类型实现。参数通常保留：

```text
Message
Subject
To
Token
```

`Token` 填企业微信机器人 key。脚本内部不要把完整 token 输出到日志，排错时应脱敏。

## 易错点

| 问题           | 处理                                              |
| ------------ | ----------------------------------------------- |
| Web 页面打不开    | 检查 httpd、php-fpm、zabbix-server、数据库是否启动          |
| 前端源不可用       | 检查 zabbix.repo 的 frontend 是否 enabled            |
| SCLo 源失效     | 替换 CentOS-SCLo-scl 和 scl-rh 源                   |
| Agent 无数据    | 检查 10050/10051、防火墙、Server/ServerActive/Hostname |
| 自定义项不生效      | 重启 agent，用 zabbix_get 测试                        |
| Nginx 状态采集为空 | 确认 stub_status 路径和 allow/deny                   |
| 报警不发送        | 从触发器、动作、媒介、用户媒介逐层排查                             |
