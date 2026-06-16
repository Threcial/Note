---
type: knowledge
status: evergreen
created: 2026-06-13
updated: 2026-06-13
related_to:
  - "[[Linux 运维基础]]"
  - "[[CentOS 7]]"
has:
  - "[[ls]]"
  - "[[cd]]"
  - "[[cp]]"
  - "[[mv]]"
  - "[[rm]]"
  - "[[cat]]"
  - "[[less]]"
  - "[[find]]"
  - "[[locate]]"
  - "[[stat]]"
  - "[[date]]"
  - "[[alias]]"
_organized: true
---

# Linux 基础命令

Linux 的入口只有一个根目录 `/`，命令严格区分大小写，基本格式是 `命令 [选项] [参数]`。`root` 的提示符通常是 `#`，普通用户通常是 `$`。

## 登录提示符

```bash
[root@localhost ~]#
```

| 部分 | 含义 |
| --- | --- |
| `root` | 当前登录用户名 |
| `@` | 分隔符 |
| `localhost` | 主机名 |
| `~` | 当前用户家目录 |
| `#` | root 用户提示符 |
| `$` | 普通用户提示符 |

## 命令格式

```bash
命令 [选项] [参数]
```

长命令可以用 `\` 换行：

```bash
systemctl list-unit-files | grep \
enable
```

## 开关机命令

```bash
shutdown -h now     # 立即关机
shutdown -r now     # 立即重启
reboot now          # 立即重启
```

## 目录与文件命令

### ls

```bash
ls -alh
ls -alh --time-style=long-iso
ls -i
```

| 选项 | 含义 |
| --- | --- |
| `-a` | 显示所有文件，包括隐藏文件 |
| `-l` | 长格式显示 |
| `-h` | 人类可读大小 |
| `-i` | 显示 inode 编号 |

### pwd / tree

```bash
pwd
tree -C
tree -d
```

### cd

```bash
cd ..          # 返回上一级目录
cd -           # 返回上一次所在目录
cd ~           # 返回当前用户家目录
```

### 绝对路径与相对路径

```text
/etc/systemd/system.conf       # 绝对路径，从 / 开始
systemd/system.conf            # 相对路径，从当前目录开始
```

### touch / mkdir

```bash
touch hello.txt
mkdir -p a/b/c
mkdir -pv a/b/c
```

### rm

```bash
rm -f 文件
rm -rf 目录
```

> `rm -rf` 删除后通常无法直接恢复。生产环境删除前应确认路径、确认备份。

### mv

```bash
mv hello /app/          # 移动文件
mv aaa kkk              # 重命名
mv han/* app/           # 只移动目录内容
```

### cp

```bash
cp -a hello /app/       # 归档复制，保留属性
cp -a hello/* /app/
```

## 常用快捷键

| 快捷键 | 作用 |
| --- | --- |
| `Tab` | 命令、文件、目录补全 |
| `Ctrl+a` | 光标到行首 |
| `Ctrl+e` | 光标到行尾 |
| `Ctrl+l` | 清屏 |
| `Ctrl+c` | 结束当前命令 |

## 查看文件内容

```bash
cat -n 文件
head -n 20 文件
tail -n 20 文件
tail -f 文件
```

### less

```bash
less -N 文件
```

| less 操作 | 含义 |
| --- | --- |
| 空格 | 向下翻页 |
| `b` | 向上翻页 |
| `/字符串` | 向后搜索 |
| `?字符串` | 向前搜索 |
| `q` | 退出 |

### man

```bash
man ls
man systemctl
```

## 时间与时区

```bash
timedatectl
date "+%Y-%m-%d %H:%M:%S"
```

## history

```bash
history 20
```

## alias 别名

```bash
alias
alias 别名='命令'
unalias 别名
```

绕过别名：

```bash
/bin/rm 文件
\rm 文件
```

## file 查看文件类型

```bash
file /bin/cat
```

常见输出：`ASCII text`、`ELF 64-bit LSB executable`、`directory`、`symbolic link` 等。

## ls -l 文件属性

```text
-rwxrwxrwx 1 root root 25 Dec 17 10:06 hello.py
```

列含义：文件类型和权限、硬链接数、属主、属组、文件大小、文件时间、文件名。

## stat 与文件三种时间

```bash
stat test.txt
```

| 时间 | 含义 |
| --- | --- |
| Access / atime | 最近访问时间 |
| Modify / mtime | 文件内容最近修改时间 |
| Change / ctime | 文件元数据最近变更时间 |

`ls -l` 默认显示 `mtime`。

## which 与 whereis

```bash
which cp
whereis ls
```

## locate

```bash
locate 文件名
updatedb
```

基于数据库查找，速度快但不是实时更新。

## find

```bash
find 要查找的目录 查找条件 查找到后的动作
```

### 常用条件

| 条件 | 含义 |
| --- | --- |
| `-name` | 按文件名查找 |
| `-perm` | 按权限查找 |
| `-user` | 按属主查找 |
| `-mtime -n/+n` | n 天以内/以前 |
| `-type` | 按文件类型 |
| `-size` | 按大小 |
| `-maxdepth num` | 限制递归深度 |

### 文件类型

| 类型 | 含义 |
| --- | --- |
| `d` | 目录 |
| `f` | 普通文件 |
| `l` | 符号链接 |
| `s` | socket |

### 动作

| 动作 | 含义 |
| --- | --- |
| `-print` | 输出匹配结果，默认 |
| `-exec 命令 {} \;` | 对每个结果执行命令 |

### 示例

```bash
find . -name "*.c"
find /home -size +1M
find /var/log -mtime +7
find . -type f -perm 644 -exec ls -l {} \;
```

## 命令速查

| 目标 | 命令 |
| --- | --- |
| 当前路径 | `pwd` |
| 创建空文件 | `touch file` |
| 创建目录 | `mkdir dir` |
| 递归创建目录 | `mkdir -p a/b/c` |
| 复制并保留属性 | `cp -a src dst` |
| 分页查看 | `less file` |
| 实时跟踪 | `tail -f file` |
| 查看类型 | `file file` |
| 查命令路径 | `which cmd` |
| 查文件位置 | `find 路径 条件` |

## 易错点

| 场景 | 处理方式 |
| --- | --- |
| 把 `/tmp` 当长期存储目录 | 不要保存重要文件，可能被清理 |
| `rm -rf *` 路径没确认 | 执行前先 `pwd`、`ls` 确认当前目录 |
| `cp` / `mv` 覆盖同名文件 | 先检查目标目录 |
| `find /` 全盘查找 | 尽量缩小路径范围 |
