---
type: knowledge
status: evergreen
created: 2026-06-13
updated: 2026-06-13
related_to:
  - "[[Linux 运维基础]]"
  - "[[服务管理]]"
has:
  - "[[systemctl]]"
  - "[[systemd]]"
_organized: true
---

# systemd 服务管理

CentOS 7 以 `systemd` 为主，`systemctl` 是主要管理入口。

## 运行级别与 target

| 运行级别 | systemd target | 含义 |
| --- | --- | --- |
| 0 | `poweroff.target` | 关机 |
| 1 | `rescue.target` | 单用户/救援模式 |
| 3 | `multi-user.target` | 多用户字符模式 |
| 5 | `graphical.target` | 图形模式 |
| 6 | `reboot.target` | 重启 |

```bash
systemctl get-default
systemctl set-default multi-user.target
```

## 服务单元文件目录

| 路径 | 含义 | 优先级 |
| --- | --- | --- |
| `/usr/lib/systemd/system/` | 软件包安装的服务单元 | 低 |
| `/run/systemd/system/` | 运行时生成的服务单元 | 中 |
| `/etc/systemd/system/` | 管理员自定义或覆盖 | 高 |

## systemctl 常用动作

```bash
systemctl start httpd
systemctl stop httpd
systemctl restart httpd
systemctl reload httpd
systemctl status httpd
systemctl enable httpd
systemctl disable httpd
systemctl is-enabled httpd
systemctl daemon-reload
```

`restart` 会重启进程，`reload` 只重载配置。

## systemctl status 关键字段

| 字段 | 含义 |
| --- | --- |
| `Loaded` | 是否加载、是否开机启动 |
| `Active` | 当前运行状态 |
| `Main PID` | 主进程号 |
| `Memory` | 当前占用内存 |

常见状态：`active (running)`、`active (exited)`、`inactive (dead)`、`failed`。

## 查看日志

```bash
journalctl -u httpd
journalctl -u httpd -f
journalctl -xe
```

## init 与 systemd 对比

CentOS 6 旧方式：

```bash
/etc/init.d/servername start
chkconfig --level 3 servername on
```

CentOS 7 之后：

```bash
systemctl start servername
systemctl enable servername
```

## rc.local

CentOS 7 中 `rc.local` 默认可能没有执行权限，已不推荐作为长期方案。开机自启脚本更适合制作成 systemd service。

## 易错点

| 问题 | 说明 |
| --- | --- |
| `restart` 与 `reload` 不同 | `restart` 重启进程，`reload` 只重新加载配置 |
| 修改 unit 后未 daemon-reload | 修改不会立即生效 |
| 依赖 `rc.local` 开机自启 | CentOS 7 更推荐 systemd service |
