---
type: knowledge
status: evergreen
created: 2026-06-13
updated: 2026-06-13
related_to:
  - "[[linux-网络命令与排错]]"
  - "[[远程运维]]"
has:
  - "[[ssh-keygen]]"
  - "[[ssh-copy-id]]"
  - "[[rsync]]"
_organized: true
---

# 远程连接与文件同步

## SSH 密钥登录

### 生成密钥

```bash
ssh-keygen
```

默认生成 `~/.ssh/id_rsa`（私钥）和 `~/.ssh/id_rsa.pub`（公钥）。

### 发送公钥

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub root@远程主机IP
```

### SSH 配置

```bash
vim /etc/ssh/sshd_config
```

```conf
PubkeyAuthentication yes
PasswordAuthentication no
GSSAPIAuthentication no
UseDNS no
```

修改后验证并重启：

```bash
sshd -t
systemctl restart sshd
```

> 正式禁用密码登录前，必须确认密钥登录已经可以正常使用，并保留当前 SSH 会话。

## rsync 远程同步

rsync 使用增量复制，通过 SSH 传输。

### 常用参数

| 参数 | 含义 |
| --- | --- |
| `-a` | 归档模式，保持属性 |
| `-v` | 显示详细过程 |
| `-z` | 传输时压缩 |
| `-P` | 显示进度并保留部分传输文件 |
| `--delete` | 删除目标端多余文件 |
| `--exclude` | 排除文件或目录 |
| `--dry-run` | 预演，不实际修改 |

### 拉取与推送

```bash
# 拉取
rsync -av root@远程IP:/etc/hosts /tmp

# 推送
rsync -av /tmp/1.txt root@远程IP:/tmp
```

### 目录末尾斜杠

```bash
rsync -av /new 目标/     # 同步 new 目录本身
rsync -av /new/ 目标/    # 只同步 new 里的内容
```

### 排除文件

```bash
rsync -avzP --exclude-from=exclude.txt 源 目标
```

## 易错点

| 问题 | 说明 |
| --- | --- |
| 禁用 SSH 密码登录前没验证密钥 | 容易造成无法远程登录 |
| rsync 目录末尾 `/` | 会影响是否同步目录本身 |
| `--delete` 使用前未预演 | 会删除目标端多余文件，先 `--dry-run` |
