---
type: knowledge
status: evergreen
created: 2026-06-13
updated: 2026-06-13
related_to:
  - "[[linux-基础命令]]"
  - "[[文本处理]]"
has:
  - "[[sort]]"
  - "[[uniq]]"
  - "[[wc]]"
  - "[[grep]]"
  - "[[sed]]"
  - "[[awk]]"
  - "[[正则表达式]]"
_organized: true
---

# 文本处理与正则表达式

## sort

```bash
sort [选项] 文件
```

| 参数 | 含义 |
| --- | --- |
| `-c` | 检查是否已排序 |
| `-t` | 指定分隔符 |
| `-k` | 指定排序字段 |
| `-n` | 按数值排序 |
| `-r` | 反向排序 |
| `-o` | 输出到指定文件 |

```bash
sort -t ':' -k 3 -n passwd
sort passwd -o passwd     # 写回原文件必须用 -o
```

## uniq

```bash
sort 文件 | uniq
```

| 参数 | 含义 |
| --- | --- |
| `-c` | 显示重复次数 |
| `-d` | 只显示重复行 |
| `-u` | 只显示只出现一次的行 |

`uniq` 只能识别相邻重复行，不排序时可能漏统计。

## wc

```bash
wc -l 文件    # 行数
wc -w 文件    # 单词数
wc -m 文件    # 字符数
```

## grep

```bash
grep [选项] '模式' 文件
```

| 参数 | 含义 |
| --- | --- |
| `-v` | 取反 |
| `-i` | 忽略大小写 |
| `-n` | 显示行号 |
| `-w` | 按单词匹配 |
| `-o` | 只显示匹配到的内容 |
| `-E` | 使用扩展正则表达式 |
| `-r` | 递归查找目录 |

```bash
grep -n 'linux' passwd
grep -E 'linux|weibo' passwd
```

## 正则表达式

### 基础正则 BRE

| 表达式 | 含义 |
| --- | --- |
| `^` | 行首 |
| `$` | 行尾 |
| `.` | 任意一个字符 |
| `*` | 前一个字符重复 0 次或多次 |
| `.*` | 任意长度内容 |
| `[abc]` | 匹配集合中任意一个字符 |

### 扩展正则 ERE（需 `grep -E`）

| 表达式 | 含义 |
| --- | --- |
| `+` | 前一个字符出现 1 次或多次 |
| `?` | 前一个字符出现 0 次或 1 次 |
| `|` | 或 |
| `{n,m}` | 最少 n 次、最多 m 次 |
| `()` | 分组 |

## sed

```bash
sed [选项] '动作' 文件
```

| 参数 | 含义 |
| --- | --- |
| `-n` | 取消默认输出，常与 `p` 配合 |
| `-i` | 直接修改文件 |
| `-r` | 使用扩展正则 |

| 动作 | 含义 |
| --- | --- |
| `p` | 打印 |
| `d` | 删除 |
| `s` | 替换 |
| `a` | 在指定行后追加 |
| `i` | 在指定行前插入 |

```bash
sed -n '2,3p' linux.txt
sed 's#linux#windows#g' linux.txt
sed -i.bak 's#old#new#g' file
```

### 实战：取网卡 IP

```bash
ip -o -4 addr show dev ens33 | awk '{print $4}' | cut -d/ -f1
```

### 实战：取文件权限

```bash
stat -c '%a' /etc/hosts
```

## awk

```bash
awk [选项] '条件 {动作}' 文件
```

| 参数/变量 | 含义 |
| --- | --- |
| `-F` | 指定分隔符 |
| `$1` | 第一列 |
| `$0` | 整行 |
| `$NF` | 最后一列 |
| `NR` | 当前行号 |

```bash
awk -F ":" '{print $3,$5}' test.txt
awk 'NR==3' passwd
awk '/lp/' test.txt
awk '$NF>70 {print $0}' a.txt
```

## grep、sed、awk 的分工

| 工具 | 适合做什么 |
| --- | --- |
| `grep` | 找行、过滤行 |
| `sed` | 按行编辑、替换、删除、插入 |
| `awk` | 按列处理、格式化输出、简单计算 |

```bash
grep 'error' app.log | awk '{print $1,$2,$NF}'
```

## 命令速查

| 目标 | 命令 |
| --- | --- |
| 按数值排序 | `sort -t':' -k3 -n passwd` |
| 去重统计 | `sort file \| uniq -c` |
| 统计行数 | `wc -l file` |
| 查找行 | `grep 'pattern' file` |
| 替换文本 | `sed 's/old/new/g' file` |
| 取第 n 列 | `awk '{print $n}' file` |

## 易错点

| 问题 | 说明 |
| --- | --- |
| `uniq` 不先排序 | 只能处理相邻重复行 |
| `sort > 原文件` | 可能清空原文件，用 `sort -o` |
| Shell 通配符和正则混用 | `*` 在通配符和正则中含义不同 |
| `sed -i` 直接改文件 | 建议先不加 `-i` 验证 |
| awk 默认分隔符 | 默认按空白分割，处理 `/etc/passwd` 用 `-F ":"` |
