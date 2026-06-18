---
type: Reference
status: Organized
Belongs to: "[[Linux]]"
Related to:
  - "[[linux-进程管理]]"
Has:
  - "[[nohup]]"
  - "[[jobs]]"
  - "[[uptime]]"
  - "[[free]]"
  - "[[iftop]]"
  - "[[iostat]]"
source: ""
created: 2026-06-13
updated: 2026-06-13
---

# 后台任务与系统监控

## 后台运行程序

在命令末尾加 `&` 把程序放到后台运行：

```bash
ping www.qq.com &
ping www.qq.com >>11.txt 2>&1 &
```

查看当前 Shell 的后台任务：

```bash
jobs -l
fg %1             # 恢复到前台
kill PID          # 结束后台任务
```

## nohup

`nohup` 让进程忽略 HUP 信号，退出终端后进程继续运行：

```bash
nohup ping www.qq.com >>11.txt 2>&1 &
```

| 部分 | 作用 |
| --- | --- |
| `nohup` | 忽略 HUP 信号 |
| `>>11.txt 2>&1` | 输出写入文件 |
| `&` | 放入后台运行 |

长期服务更适合写成 systemd service，而不是长期依赖 nohup。

## 系统信息

### uptime

```bash
uptime            # 当前时间、运行时长、登录用户数、平均负载
uptime -p
uptime -s
```

负载需要结合 CPU 核心数判断，不能只看数值大小。

### free

```bash
free -h
free -s 5         # 每 5 秒刷新
```

`available` 通常比 `free` 更适合判断可用内存。

### lscpu

```bash
lscpu
lscpu -e          # CPU 编号、核心、Socket
```

### last

```bash
last
```

## 网络与磁盘监控

### iftop

```bash
iftop -Bnt -i ens33
```

| 参数 | 含义 |
| --- | --- |
| `-i` | 指定网卡 |
| `-n` | 不解析域名 |
| `-t` | 文本输出 |
| `-B` | 以 Byte 为单位 |

### iostat

```bash
iostat -x 1       # 每 1 秒刷新
```

| 字段 | 含义 |
| --- | --- |
| `%iowait` | 等待 I/O 的 CPU 时间 |
| `tps` | 每秒 I/O 请求次数 |
| `kB_read/s` | 每秒读取量 |

### iotop

```bash
iotop -o          # 只显示有 I/O 的进程
```

## 易错点

| 问题 | 说明 |
| --- | --- |
| `nohup` 不等于后台运行 | 后台运行由 `&` 完成，nohup 负责忽略 HUP |
