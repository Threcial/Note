---
type: Concept
status: Organized
Belongs to: "[[Linux]]"
Has:
  - "[[系统启动流程]]"
  - "[[systemd]]"
  - "[[SELinux]]"
  - "[[Linux 目录结构]]"
source: ""
created: 2026-06-13
updated: 2026-06-13
---

# 系统启动与核心目录

## CentOS 7 启动流程

```text
POST 开机自检 → BIOS 选择启动设备 → 读取 MBR Bootloader → 加载 GRUB → 加载 kernel → initramfs 加载核心模块 → 挂载根目录 → 启动 systemd → 进入 default.target → sysinit.target → basic.target → multi-user.target / graphical.target
```

## 运行级别与 systemd target

| 运行级别 | systemd target | 含义 |
| --- | --- | --- |
| 0 | `poweroff.target` | 关机 |
| 1 | `rescue.target` | 单用户/救援模式 |
| 3 | `multi-user.target` | 多用户文本模式 |
| 5 | `graphical.target` | 图形界面 |
| 6 | `reboot.target` | 重启 |

## 用户登录时加载的文件

```bash
/etc/profile        # 全局环境变量
/etc/profile.d/     # 全局脚本目录
~/.bashrc           # 当前用户 Shell 配置
```

## /etc 核心文件

| 路径 | 作用 |
| --- | --- |
| `/etc/sysconfig/network-scripts/ifcfg-ens33` | 网卡配置文件 |
| `/etc/resolv.conf` | DNS 服务器配置 |
| `/etc/hosts` | 本地域名与 IP 映射 |
| `/etc/fstab` | 开机自动挂载配置 |
| `/etc/sysctl.conf` | 内核参数配置 |
| `/etc/redhat-release` | 系统版本信息 |
| `/etc/hostname` | 主机名 |

## /usr 核心目录

| 路径 | 作用 |
| --- | --- |
| `/usr/local` | 编译安装软件的默认位置 |
| `/usr/src` | 源码存放路径 |

## /var 核心目录

| 路径 | 作用 |
| --- | --- |
| `/var/log` | 系统和服务日志目录 |
| `/var/log/messages` | 系统级日志 |
| `/var/log/secure` | 用户登录、安全认证日志 |

## /proc 核心文件

| 路径 | 作用 | 对应命令 |
| --- | --- | --- |
| `/proc/meminfo` | 内存信息 | `free -h` |
| `/proc/cpuinfo` | CPU 信息 | `lscpu` |
| `/proc/loadavg` | 系统负载 | `uptime` |

## 主机名查看与更改

```bash
hostnamectl
hostnamectl set-hostname didiyun
```

## SELinux

SELinux 是内核安全模块，提供强制访问控制。

```bash
getenforce
setenforce 0              # 临时关闭
```

永久关闭：`/etc/selinux/config` 中设置 `SELINUX=disabled`。

> 生产环境不应简单把 SELinux 当成必须关闭的东西，优先确认策略和安全要求。

## Shell 通配符

### `*`

匹配任意字符（默认不匹配隐藏文件）：

```bash
cp * /tmp/
ls *.txt
```

### `?`

匹配任意单个字符：

```bash
ls ?.txt
```

### `[]`

匹配中括号中任意一个字符：

```bash
ls [a-z].txt
ls [abc].txt
```

`[test].txt` 匹配 `t.txt`、`e.txt`、`s.txt` 中的一个，不是匹配 `test.txt`。

### `[!...]` 与 `[^...]`

取反匹配。

## 其他 Shell 特殊符号

| 符号 | 含义 |
| --- | --- |
| `;` | 命令分隔符，前一个失败不影响后续 |
| `\` | 转义，让特殊字符还原字面含义 |
| `{1..6}` | 生成序列 |
| `&&` | 前一个命令成功才执行后一个 |
| `\|\|` | 前一个命令失败才执行后一个 |

## 易错点

| 场景 | 说明 |
| --- | --- |
| `*` 匹配隐藏文件 | 默认不匹配隐藏文件 |
| `[test]` 匹配字符串 | 实际只匹配其中任意一个字符 |
| 依赖 `rc.local` 开机自启 | CentOS 7 更推荐 systemd service |
| 修改 `/etc/fstab` 未验证 | 可能导致启动失败 |
