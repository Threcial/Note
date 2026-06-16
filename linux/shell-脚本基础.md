---
type: knowledge
status: evergreen
created: 2026-06-13
updated: 2026-06-13
related_to:
  - "[[Shell 编程]]"
  - "[[linux-文本处理与正则表达式]]"
has:
  - "[[bash]]"
  - "[[Shell 变量]]"
  - "[[环境变量]]"
  - "[[位置参数]]"
  - "[[exit]]"
  - "[[read]]"
  - "[[test]]"
  - "[[if]]"
  - "[[case]]"
  - "[[while]]"
  - "[[for]]"
_organized: true
---

# Shell 脚本基础

Shell 是命令解释器，CentOS 7 默认 Bash。脚本第一行写 shebang：`#!/bin/bash`。

## 执行方式

| 执行方式 | 是否需要执行权限 | 是否新建子 Shell | 是否影响当前变量 |
| --- | --- | --- | --- |
| `./hello.sh` | 需要 | 是 | 否 |
| `bash hello.sh` | 不需要 | 是 | 否 |
| `source hello.sh` | 不需要 | 否 | 是 |

## 变量

### 赋值规则

```bash
name="cjk"
now=$(date)
test="${test}456"
```

| 规则 | 说明 |
| --- | --- |
| 变量名 | 字母、数字、下划线，不能以数字开头 |
| 等号 | `=` 两边不能有空格 |
| 值中有空格 | 必须用引号包起来 |
| 命令结果赋值 | 推荐 `$()` |

### 单引号与双引号

单引号不解析变量，双引号会解析：

```bash
abc='hello,${bn}'    # 保留字面
abc="hello,${bn}"    # 解析变量
```

建议默认使用双引号包裹变量：`echo "${name}"`。

### 数字运算

Bash 变量默认按字符串处理：

```bash
num3=$((num1 + num2))
let num=1+20
```

### 环境变量

```bash
env
echo "$PATH"
export PATH="/opt/bin:$PATH"
```

常见变量：`HOSTNAME`、`SHELL`、`USER`、`PATH`、`HOME`、`LANG`。

### 特殊变量

| 变量 | 含义 |
| --- | --- |
| `$0` | 脚本名 |
| `${1}` | 第一个参数 |
| `$#` | 参数总数 |
| `$@` | 所有参数 |
| `$?` | 上一个命令退出状态 |
| `$$` | 当前 Shell PID |

### `$?` 与 exit

```bash
exit 0    # 成功
exit 1    # 失败或异常
```

### dirname 与 basename

```bash
dirname "$0"    # /root
basename "$0"   # bb.sh
```

### shift

```bash
shift     # 位置参数左移一次
```

### read

```bash
read -r -p "input: " value
```

`-r` 避免反斜杠被转义。

## 条件表达式

```bash
[ 条件表达式 ]
[[ 条件表达式 ]]    # Bash 扩展，支持 &&、||、正则
```

### 数字比较

| 操作符 | 含义 |
| --- | --- |
| `-eq` | 等于 |
| `-ne` | 不等于 |
| `-gt` | 大于 |
| `-ge` | 大于等于 |
| `-lt` | 小于 |
| `-le` | 小于等于 |

### 字符串比较

```bash
[ "${name}" == "root" ]
```

### 文件判断

| 操作符 | 含义 |
| --- | --- |
| `-d` | 是否是目录 |
| `-f` | 是否是普通文件 |
| `-r` | 是否可读 |
| `-w` | 是否可写 |
| `-x` | 是否可执行 |
| `-s` | 文件大小是否非 0 |
| `-n` | 字符串长度是否非 0 |
| `-z` | 字符串长度是否为 0 |

## if

```bash
if [ 条件 ]; then
    代码
elif [ 条件 ]; then
    代码
else
    代码
fi
```

```bash
if [ $# -ne 2 ]; then
    echo "please input two args"
fi
```

## case

```bash
case "${age}" in
    10) echo "age is 10" ;;
    30) echo "age is 30" ;;
    *) echo "age error" ;;
esac
```

## while

```bash
while [ 条件 ]
do
    代码
done
```

无限循环：

```bash
while true; do uptime; sleep 2; done
```

### while 读取文件

推荐写法：

```bash
while IFS= read -r line
do
    echo "${line}"
done < "$1"
```

## break 与 continue

```bash
break       # 跳出循环
continue    # 跳过本轮剩余代码
```

## for

```bash
for x in 22 "abc" 66 "nihao"
do
    echo "${x}"
done

for ((i=0; i<4; i++))
do
    echo "${i}"
done
```

## 脚本编写建议

| 建议 | 说明 |
| --- | --- |
| 使用 `#!/bin/bash` | 明确解释器 |
| 变量加双引号 | 避免空格、通配符问题 |
| 复杂命令先手工测试 | 再写进脚本 |
| 参数先校验 | 再执行危险操作 |
| 必要时加调试选项 | `bash -x script.sh` |

## 易错点

| 问题 | 说明 |
| --- | --- |
| 数字比较用 `>`、`<` | 可能按字符串比较或被当成重定向 |
| `for x in $(ls)` 遇空格 | 无法安全处理带空格的文件名 |
| `$?` 对 `expr 0` 判断整数 | `expr` 结果为 0 时 `$?` 可能返回 1 |
| 函数里用 `return` 想退出脚本 | `return` 只退出函数，退出脚本用 `exit` |
