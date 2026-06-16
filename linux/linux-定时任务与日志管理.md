---
type: knowledge
status: evergreen
created: 2026-06-13
updated: 2026-06-13
related_to:
  - "[[Linux 运维基础]]"
  - "[[任务自动化]]"
  - "[[日志管理]]"
has:
  - "[[/dev/null]]"
  - "[[crond]]"
  - "[[crontab]]"
  - "[[logrotate]]"
  - "[[gzip]]"
_organized: true
---

# 定时任务与日志管理

## /dev/null

黑洞设备，写入其中的内容会被丢弃：

```bash
ntpdate ntp3.aliyun.com >/dev/null 2>&1
```

`$?` 返回 `0` 表示成功，非 `0` 表示失败。

## crond 与 crontab

| 名称 | 含义 |
| --- | --- |
| `crond` | 服务进程，真正执行任务 |
| `crontab` | 设置用户定时任务规则的命令 |

### 服务管理

```bash
systemctl status crond
crontab -l          # 查看当前用户任务
crontab -e          # 编辑当前用户任务
```

用户任务存放位置：`/var/spool/cron`。

### 时间格式

```text
*  *  *  *  *  command
分 时 日 月 周  命令
```

| 符号 | 含义 |
| --- | --- |
| `*` | 任意时间 |
| `-` | 时间范围 |
| `,` | 离散时间点 |
| `/n` | 每隔 n 个单位 |

### 示例

```cron
00 21 * * * cmd                    # 每天 21:00
*/2 * * * * cmd                    # 每 2 分钟
00 8,13,17,21,23 * * * cmd        # 每天指定整点
30 */6 * * * cmd                   # 每 6 小时的半点
45 4 1,10,22 * * cmd               # 每月 1、10、22 日 4:45
10 10 * * 6,0 cmd                  # 每周六、周日 10:10
```

### 编写要领

| 要领 | 原因 |
| --- | --- |
| 定时任务要写日志 | 方便确认是否执行、执行结果 |
| 复杂任务放脚本执行 | crontab 中只保留调度入口 |
| 命令尽量写绝对路径 | cron 环境变量很少 |
| 结尾加 `>/dev/null 2>&1` | 避免无意义输出堆积 |
| 先命令行测试，再脚本，再 crontab | 降低排错成本 |

### 实战：定时备份

```bash
#!/bin/bash
cd /var/www
/usr/bin/tar -zcf /opt/www_`date +%F-%T`.tar.gz ./html
```

crontab 中 `%` 需要转义：

```cron
00 0 * * * cd /var/www && /usr/bin/tar -zcf /opt/www_`date +\%F-\%T`.tar.gz ./html &> /dev/null
```

## logrotate 日志切割

### 相关文件

| 路径 | 作用 |
| --- | --- |
| `/etc/logrotate.conf` | 主配置文件 |
| `/etc/logrotate.d/` | 自定义日志切割配置目录 |
| `/etc/cron.daily/logrotate` | 系统定期调用 logrotate 的脚本 |

### 主配置常见项

```text
weekly
rotate 4
create
dateext
include /etc/logrotate.d
```

### create 与 copytruncate

| 模式 | 行为 | 适用情况 |
| --- | --- | --- |
| `create` | 重命名归档，创建新文件 | 服务能重新打开日志，推荐优先 |
| `copytruncate` | 复制并截断 | 服务无法切换日志文件时使用 |

### 手动执行

```bash
logrotate -vf /etc/logrotate.conf
```

## gzip 压缩

```bash
gzip -v services            # 默认会替换源文件
gzip -c services > test.gz  # 保留源文件
gzip -tv test.gz            # 检查压缩包完整性
gzip -9cv services > test9.gz   # 最高压缩比
```

| 命令 | 作用 |
| --- | --- |
| `zless file.gz` | 直接分页查看内容 |
| `zgrep -n ssh file.gz` | 直接搜索内容 |

## 命令速查

| 目标 | 命令 |
| --- | --- |
| 丢弃全部输出 | `命令 >/dev/null 2>&1` |
| 编辑定时任务 | `crontab -e` |
| 查看 cron 日志 | `tail -f /var/log/cron` |
| 手动 logrotate | `logrotate -vf /etc/logrotate.conf` |
| gzip 保留源文件 | `gzip -c file > file.gz` |
| 搜索 gzip 内容 | `zgrep -n 字符串 file.gz` |

## 易错点

| 场景 | 说明 |
| --- | --- |
| crontab 中 `%` 不转义 | date 格式里的 `%` 要写成 `\%` |
| 定时任务使用相对路径 | cron 工作目录不稳定，应使用绝对路径 |
| 定时任务不写日志 | 失败后难以判断原因 |
| `copytruncate` 随便使用 | 存在日志丢失窗口，优先考虑 `create` |
| `gzip file` 想保留源文件 | 默认会替换源文件，用 `gzip -c file > file.gz` |
