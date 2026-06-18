---
type: Reference
status: Organized
Belongs to: "[[Linux]]"
Related to:
  - "[[linux-systemd-服务管理]]"
  - "[[进程管理]]"
Has:
  - "[[ps]]"
  - "[[top]]"
  - "[[kill]]"
  - "[[lsof]]"
source: ""
created: 2026-06-13
updated: 2026-06-13
---

# 进程管理

程序是静态指令存放在磁盘中，进程是程序运行后的活动实体，有自己的 PID 和资源上下文。

## ps

```bash
ps -ef
ps aux
ps -u root
ps -l
```

`ps aux` 字段：USER、PID、%CPU、%MEM、VSZ、RSS、TTY、STAT、START、TIME、COMMAND。

`STAT` 常见状态：

| 状态 | 含义 |
| --- | --- |
| `R` | 运行中 |
| `S` | 可唤醒睡眠 |
| `D` | 不可中断睡眠，常与 I/O 等待有关 |
| `T` | 停止 |
| `Z` | 僵尸进程 |

## pstree

```bash
pstree -p
pstree -u nobody
```

适合观察父子进程关系，例如 nginx master/worker 结构。

## top

```bash
top -d 1              # 刷新间隔 1s
top -n 3 -d 1         # 刷新 3 次后退出
top -p 7726           # 查看指定 PID
```

交互按键：`1` 显示所有 CPU 核、`M` 按内存排序、`P` 按 CPU 排序、`q` 退出。

CPU 字段：`us` 用户态、`sy` 内核态、`id` 空闲、`wa` I/O 等待、`st` 虚拟化偷取。

## pidof、pgrep、kill、killall

```bash
pidof sshd
pgrep -a sshd
```

```bash
kill -15 PID          # 正常结束（默认信号）
kill -9 PID           # 强制结束（无法被捕获）
kill -1 PID           # 重新加载配置
kill -l               # 列出所有信号
```

按进程名结束：

```bash
killall ping
pkill ping
pkill -f "python app.py"
```

## lsof

```bash
lsof -i:22            # 查看端口占用
lsof -u root          # 查看用户打开的文件
lsof -p 1             # 查看 PID 打开的文件
lsof /var/log/messages # 查看哪些进程使用该文件
```

## 系统信息查看

```bash
uptime                # 运行时长和负载
free -h               # 内存使用
lscpu                 # CPU 信息
last                  # 登录记录
```

## 易错点

| 问题 | 说明 |
| --- | --- |
| `kill -9` 不应作为默认手段 | 它不会给进程清理资源的机会 |
| 负载只看数值不看核心数 | 负载需结合 CPU 核心数判断 |
