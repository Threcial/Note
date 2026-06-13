---
title: 定时任务、日志切割与 gzip 压缩
type: knowledge
status: evergreen
created: 2026-06-13
updated: 2026-06-13
tags:
  - Linux
  - CentOS7
  - crontab
  - crond
  - logrotate
  - gzip
  - 定时任务
  - 日志切割
aliases:
  - DAY05
  - Linux 定时任务
  - Linux 日志切割
  - gzip 压缩
Belongs to:
  - "[[Linux 运维基础]]"
  - "[[任务自动化]]"
  - "[[日志管理]]"
Related to:
  - "[[day04 LVM、RAID 与 tar 打包压缩]]"
  - "[[Shell 重定向]]"
  - "[[标准输出与标准错误]]"
  - "[[Shell 脚本]]"
  - "[[PATH 环境变量]]"
  - "[[Linux 日志]]"
Has:
  - "[[/dev/null]]"
  - "[[crond]]"
  - "[[crontab]]"
  - "[[/var/log/cron]]"
  - "[[logrotate]]"
  - "[[gzip]]"
  - "[[定时备份]]"
  - "[[日志轮转]]"
_organized: true
---
# 定时任务、日志切割与 gzip 压缩

## 概览

- `/dev/null` 是黑洞设备，常用于丢弃不需要的标准输出和标准错误。
- `crond` 是执行定时任务的服务进程，`crontab` 是编辑任务规则的命令。
- 定时任务环境变量很少，命令和路径应尽量写绝对路径；复杂任务写成脚本后再由 `crontab` 调用。
- `logrotate` 用于日志切割，系统通常通过 `/etc/cron.daily/logrotate` 周期调用。
- `gzip` 默认会替换源文件；要保留源文件，应使用 `gzip -c file > file.gz`。

## 关系表

| 条目              | 上游依赖          | 下游用途                 |
| --------------- | ------------- | -------------------- |
| `/dev/null`     | 标准输出、标准错误、重定向 | 静默执行、脚本输出控制          |
| `crontab`       | 时间规则、命令、脚本    | 周期任务、备份、同步、清理        |
| `crond`         | crontab 规则    | 实际执行定时任务             |
| `/var/log/cron` | crond 执行记录    | 排查定时任务不执行            |
| `logrotate`     | 日志文件路径、轮转规则   | 日志归档、日志文件控制          |
| `gzip`          | 普通文件或 tar 包   | 单文件压缩、`.tar.gz` 组合压缩 |

## /dev/null

`/dev/null` 是特殊设备文件，写入其中的内容会被丢弃，不占用磁盘空间；从中读取不会得到有效内容。

常用场景：不希望命令输出显示到屏幕，也不希望保存到文件。

```bash
ntpdate ntp3.aliyun.com >/dev/null 2>&1
```

含义：

| 片段           | 含义                 |
| ------------ | ------------------ |
| `>/dev/null` | 标准输出丢弃             |
| `2>&1`       | 标准错误合并到标准输出，因此也被丢弃 |

查看上一条命令是否成功：

```bash
echo $?
```

| 返回值   | 含义 |
| ----- | -- |
| `0`   | 成功 |
| 非 `0` | 失败 |

## 定时任务基础

Linux 周期任务由 cron 体系实现。

| 名称        | 含义            |
| --------- | ------------- |
| `cron`    | 定时任务软件体系      |
| `crond`   | 服务进程，真正执行任务   |
| `crontab` | 设置用户定时任务规则的命令 |

任务类型：

| 类型     | 说明                   |
| ------ | -------------------- |
| 系统定时任务 | 系统自动执行，例如日志切割        |
| 用户定时任务 | 管理员自定义，例如备份、更新、删除、监控 |

典型场景：业务高峰不能备份数据库，把备份脚本安排在凌晨执行。

## crond 服务与 crontab 命令

查看服务：

```bash
systemctl status crond
```

查看当前用户定时任务：

```bash
crontab -l
```

查看指定用户定时任务：

```bash
crontab -u wb -l
```

编辑当前用户定时任务：

```bash
crontab -e
```

用户定时任务存放位置：

```bash
/var/spool/cron
```

`crontab` 常用选项：

| 选项   | 含义   |
| ---- | ---- |
| `-l` | 查看   |
| `-e` | 编辑   |
| `-u` | 指定用户 |

## crontab 时间格式

```text
*  *  *  *  *  command
分 时 日 月 周  命令
```

| 字段    | 范围    | 含义         |
| ----- | ----- | ---------- |
| 第 1 位 | 00-59 | 分          |
| 第 2 位 | 00-23 | 时          |
| 第 3 位 | 1-31  | 日          |
| 第 4 位 | 1-12  | 月          |
| 第 5 位 | 0-6   | 周，0 通常表示周日 |
| 第 6 位 | 命令/脚本 | 要执行的动作     |

特殊符号：

| 符号   | 含义                        |
| ---- | ------------------------- |
| `*`  | 任意时间                      |
| `-`  | 时间范围，如 `17-20`            |
| `,`  | 离散时间点，如 `1,4,6,0`         |
| `/n` | 每隔 n 个单位，如 `*/10` 每 10 分钟 |

## crontab 示例

```cron
00 21 * * * cmd
00 8-15 * * * cmd
00 8,13,17,21,23 * * * cmd
*/2 * * * * cmd
20 13,21 * * * cmd
30 */6 * * * cmd
30 8-21/2 * * * cmd
31 21 * * * cmd
45 4 1,10,22 * * cmd
10 10 * * 6,0 cmd
0,30 18-21 * * * cmd
* 21,00-07/2 * * * cmd
30 09 * * 0 cmd
30 08 * * * cmd
```

解释：

| 表达式                      | 含义                      |
| ------------------------ | ----------------------- |
| `00 21 * * *`            | 每天 21:00                |
| `00 8-15 * * *`          | 每天 8 点到 15 点的整点         |
| `00 8,13,17,21,23 * * *` | 每天指定整点                  |
| `*/2 * * * *`            | 每 2 分钟                  |
| `20 13,21 * * *`         | 每天 13:20 和 21:20        |
| `30 */6 * * *`           | 每 6 小时的半点               |
| `30 8-21/2 * * *`        | 8 到 21 点之间每隔 2 小时的半点    |
| `45 4 1,10,22 * *`       | 每月 1、10、22 日 4:45       |
| `10 10 * * 6,0`          | 每周六、周日 10:10            |
| `0,30 18-21 * * *`       | 每天 18 到 21 点的 0 分和 30 分 |
| `30 09 * * 0`            | 每周日 9:30                |
| `30 08 * * *`            | 每天 8:30                 |

### `* */2 * * *` 的实际含义

```cron
* */2 * * * date >>/tmp/sg123.txt
```

这不是“从当前时间开始每 2 小时执行一次”，而是每天的 `0,2,4,6,8,10,12,14,16,18,20,22` 点内，每分钟执行一次，持续一个小时。

## 实战：每分钟覆盖写入 ls 结果

先命令行测试：

```bash
ls /etc/ > /tmp/ls.txt
```

写入定时任务：

```cron
*/1 * * * * ls /etc/ > /tmp/ls.txt
```

验证：

```bash
cat /tmp/ls.txt | wc -l
```

## 实战：每分钟追加 cal 结果

命令行测试：

```bash
cal >> /root/date.txt
cat /root/date.txt
```

定时任务：

```cron
*/1 * * * * cal >> /root/date.txt
```

验证：

```bash
tail -f /root/date.txt
```

原始记录使用 `tailf`，新系统更常用 `tail -f`。

## 实战：定时执行脚本

脚本：

```bash
vim /root/time.sh
```

```bash
#!/bin/bash
date >> /root/time.txt
```

测试：

```bash
chmod 777 /root/time.sh
/root/time.sh
cat /root/time.txt
```

写入定时任务：

```cron
*/1 * * * * /usr/bin/bash /root/time.sh
```

验证：

```bash
tail -f /root/time.txt
```

更稳妥的做法是不用 `chmod 777`，直接用 `/bin/bash /root/time.sh` 执行，并给脚本合理权限。

## 实战：每 5 分钟同步时间

命令行测试：

```bash
/usr/sbin/ntpdate ntp1.aliyun.com
date
```

写入定时任务：

```cron
*/5 * * * * /usr/sbin/ntpdate ntp1.aliyun.com >/dev/null 2>&1
```

## 实战：定时备份网站目录

需求：每天 0 点备份 `/var/www/html` 到 `/opt`，压缩包名包含时间。

命令行测试：

```bash
cd /var/www && tar -zcvf /opt/www_`date +%F-%T`.tar.gz html
```

直接写入 crontab 时，`%` 需要转义：

```cron
00 0 * * * cd /var/www && /usr/bin/tar -zcf /opt/www_`date +\%F-\%T`.tar.gz ./html &> /dev/null
```

脚本方式：

```bash
vim /root/backup.sh
```

```bash
#!/bin/bash
cd /var/www &&
/usr/bin/tar -zcf /opt/www_`date +%F-%T`.tar.gz ./html
```

测试：

```bash
/bin/bash /root/backup.sh
```

写入定时任务：

```cron
*/1 * * * * /bin/bash /root/backup.sh &> /dev/null
```

验证：

```bash
ls -t /opt
```

## 定时任务编写要领

| 要领                         | 原因                        |
| -------------------------- | ------------------------- |
| 定时任务要写日志                   | 方便确认是否执行、执行结果如何           |
| 复杂任务放脚本执行                  | crontab 中只保留调度入口          |
| 用 `/bin/bash` 执行脚本         | 避免忘记加执行权限导致失败             |
| 结尾加 `>/dev/null 2>&1` 或写日志 | 避免无意义输出堆积                 |
| 不要随意打印大量输出                 | 例如 tar 的 `-v` 在定时任务中通常没必要 |
| 先命令行测试，再脚本测试，最后写 crontab   | 降低排错成本                    |
| 命令尽量写绝对路径                  | cron 环境变量很少               |
| 查看 `/var/log/cron`         | 排查任务是否被调度                 |
| 简单任务可直接写 crontab           | 如时间同步                     |
| 大多数任务应写成 Shell 脚本          | 便于维护、复用、记录日志              |

如果脚本中不想每条命令都写绝对路径，可以在脚本开头显式设置环境变量：

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
```

## logrotate 日志切割

### 为什么需要日志切割

日志会随着系统运行不断增长。过大的日志文件会增加排错、备份和传输成本。`logrotate` 用于定期切割、归档、压缩和清理日志。

### 相关文件

| 路径                                    | 作用                   |
| ------------------------------------- | -------------------- |
| `/etc/logrotate.conf`                 | 主配置文件                |
| `/etc/logrotate.d/`                   | 自定义日志切割配置目录          |
| `/etc/cron.daily/logrotate`           | 系统定期调用 logrotate 的脚本 |
| `/var/lib/logrotate/logrotate.status` | logrotate 状态文件       |

### 主配置常见项

```text
weekly
rotate 4
create
create 0664 root utmp
dateext
include /etc/logrotate.d
```

| 配置                         | 含义          |
| -------------------------- | ----------- |
| `weekly`                   | 每周切割        |
| `rotate 4`                 | 保留 4 个旧日志   |
| `create`                   | 切割后创建新日志文件  |
| `create 0664 root utmp`    | 新文件权限、属主、属组 |
| `dateext`                  | 旧日志文件名包含日期  |
| `include /etc/logrotate.d` | 读取自定义配置目录   |

### 配置格式

```javascript
/var/log/cron
/var/log/maillog
/var/log/messages
/var/log/secure
/var/log/spooler
{
    missingok
    sharedscripts
    postrotate
        /bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true
    endscript
}
```

| 配置              | 含义               |
| --------------- | ---------------- |
| `missingok`     | 文件不存在不报错，继续处理下一个 |
| `sharedscripts` | 多个日志共享脚本执行逻辑     |
| `postrotate`    | 切割后执行的脚本         |
| `endscript`     | 脚本结束标记           |
| `notifempty`    | 日志为空时不轮转         |

### create 与 copytruncate

| 模式             | 行为                    | 适用情况               |
| -------------- | --------------------- | ------------------ |
| `create`       | 当前日志重命名归档，再创建新日志文件    | 服务能重新打开日志文件，推荐优先考虑 |
| `copytruncate` | 复制当前日志内容到归档文件，再截断当前日志 | 服务无法切换日志文件时使用      |

`copytruncate` 在复制和截断之间可能丢少量日志。日志量大的服务优先考虑 `create + 重载服务日志句柄`。

### 手动执行 logrotate

```bash
logrotate -vf /etc/logrotate.conf
```

| 选项   | 含义     |
| ---- | ------ |
| `-v` | 显示详细信息 |
| `-f` | 强制切割   |

### 系统如何调用 logrotate

```bash
cat /etc/cron.daily/logrotate
```

核心逻辑：

```bash
/usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf
```

如果执行失败，会调用 `logger` 记录标签为 `logrotate` 的日志。

## gzip 压缩包管理

### 命令格式

```bash
gzip [选项] 文件
```

| 选项   | 含义             |
| ---- | -------------- |
| `-c` | 输出到标准输出，不修改源文件 |
| `-d` | 解压             |
| `-t` | 检查压缩包是否损坏      |
| `-v` | 显示过程           |
| `-1` | 最快，压缩比低        |
| `-6` | 默认压缩级别         |
| `-9` | 最慢，压缩比高        |

### 默认压缩会替换源文件

```bash
gzip -v services
```

执行后 `services` 会变为 `services.gz`。

### 保留源文件

```bash
gzip -c services > test.gz
```

### 检查压缩包

```bash
gzip -tv test.gz
```

正常输出包含：

```typescript
test.gz: OK
```

### 指定压缩级别

```bash
gzip -9cv services > test9.gz
```

压缩级别越高，速度越慢，压缩比通常越高。

### 查看和搜索 gzip 内容

```bash
zless test9.gz
zgrep -n ssh test9.gz
```

| 命令      | 作用               |
| ------- | ---------------- |
| `zless` | 直接分页查看 gzip 文件内容 |
| `zgrep` | 直接搜索 gzip 文件内容   |

## 命令速查

| 目标            | 命令                                  |
| ------------- | ----------------------------------- |
| 丢弃全部输出        | `命令 >/dev/null 2>&1`                |
| 查看 cron 服务    | `systemctl status crond`            |
| 查看当前用户定时任务    | `crontab -l`                        |
| 编辑定时任务        | `crontab -e`                        |
| 查看 cron 日志    | `tail -f /var/log/cron`             |
| 手动 logrotate  | `logrotate -vf /etc/logrotate.conf` |
| gzip 压缩并保留源文件 | `gzip -c file > file.gz`            |
| gzip 检查       | `gzip -tv file.gz`                  |
| 查看 gzip 内容    | `zless file.gz`                     |
| 搜索 gzip 内容    | `zgrep -n 字符串 file.gz`              |

## 易错点

| 场景                  | 说明                                    |
| ------------------- | ------------------------------------- |
| crontab 中 `%` 不转义   | 会被 cron 特殊处理，date 格式里的 `%` 要写成 `\%`   |
| 定时任务里使用相对路径         | cron 工作目录和环境变量不稳定，应使用绝对路径             |
| 定时任务不写日志            | 失败后难以判断原因                             |
| `tar -v` 放在定时任务中    | 会产生大量无意义输出                            |
| `crond` 启动但任务没执行    | 需要查 `/var/log/cron`、脚本路径、权限、环境变量      |
| `copytruncate` 随便使用 | 存在日志丢失窗口，优先考虑 `create` 配合服务重载         |
| `gzip file` 想保留源文件  | 默认会替换源文件，应使用 `gzip -c file > file.gz` |

## 关联条目

- [[Shell 重定向]]
- [[标准输出与标准错误]]
- [[crontab]]
- [[crond]]
- [[logrotate]]
- [[gzip]]
- [[tar]]
- [[Linux 日志]]
- [[Shell 脚本]]
