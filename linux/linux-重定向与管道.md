---
type: Concept
status: Organized
Belongs to: "[[Linux]]"
Related to:
  - "[[linux-基础命令]]"
  - "[[Shell 重定向]]"
source: ""
created: 2026-06-13
updated: 2026-06-13
---

# 重定向与管道

## 标准流

| 编号 | 名称 | 含义 | 常用符号 |
| --- | --- | --- | --- |
| `0` | stdin | 标准输入 | `<`、`<<` |
| `1` | stdout | 标准输出 | `>`、`>>` |
| `2` | stderr | 标准错误 | `2>`、`2>>` |

## 覆盖与追加

```bash
命令 > out.log           # 标准输出覆盖写入
命令 >> out.log          # 标准输出追加写入
命令 2> err.log          # 标准错误覆盖写入
命令 2>> err.log         # 标准错误追加写入
命令 > all.log 2>&1      # 标准输出和标准错误写到同一文件
```

`<<` 是 Here Document，把后续多行内容作为标准输入传给命令。

## 管道

管道 `|` 把左边命令的标准输出交给右边命令处理：

```bash
ls -al /etc | less
```

管道传递的是文本流，右侧命令必须能从标准输入读取数据。

### xargs

有些命令不从标准输入读取目标，需要用 `xargs` 把输入文本转换成参数：

```bash
find /test -type f | xargs ls -l
```

对比：

```bash
find /test -type f | ls -l    # 不是想要的效果
```

## 易错点

| 场景 | 说明 |
| --- | --- |
| 把管道理解成传文件 | 管道传的是文本流 |
| `find ... | ls -l` 而非 `xargs` | `ls` 不会读管道输入 |
| `>` 覆盖文件 | 追加用 `>>`，覆盖前确认备份 |
