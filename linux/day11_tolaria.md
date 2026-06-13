---
title: Shell 脚本基础、变量、条件判断与循环
type: knowledge
status: evergreen
source: day11.md
created: 2026-06-13
updated: 2026-06-13
tags:
  - Linux
  - Shell
  - Bash
  - 变量
  - 条件判断
  - if
  - case
  - while
  - for
aliases:
  - DAY11
  - Shell 编程基础
  - Bash 变量
  - Shell 条件判断
Belongs to:
  - "[[Shell 编程]]"
  - "[[Linux 运维基础]]"
Related to:
  - "[[DAY10 - CentOS 7 安装初始化与文本处理工具]]"
  - "[[Shell 重定向]]"
  - "[[crontab]]"
  - "[[grep]]"
  - "[[sed]]"
  - "[[awk]]"
Has:
  - "[[bash]]"
  - "[[Shell 变量]]"
  - "[[环境变量]]"
  - "[[位置参数]]"
  - "[[exit]]"
  - "[[read]]"
  - "[[test 条件表达式]]"
  - "[[if]]"
  - "[[case]]"
  - "[[while]]"
  - "[[for]]"
  - "[[break]]"
  - "[[continue]]"
---

# Shell 脚本基础、变量、条件判断与循环

## 去重说明

`day10` 已经整理文本三剑客和正则表达式。本条目不重复 `grep/sed/awk` 的基础用法，只保留 Shell 脚本本身：执行方式、变量、环境变量、位置参数、条件表达式、分支和循环。

## 核心结论

| 主题 | 结论 |
| --- | --- |
| Shell | Shell 是命令解释器，CentOS 7 默认常用 Bash |
| 脚本 | 脚本是按顺序执行的文本命令集合 |
| 变量 | Bash 变量默认按字符串处理，做数学运算需要显式使用算术语法 |
| 执行方式 | `bash script.sh`、`./script.sh`、`source script.sh` 的进程环境不同 |
| 条件判断 | 数字比较、字符串比较、文件判断要使用不同操作符 |
| 循环 | `while` 适合按条件循环，`for` 适合遍历列表 |

## Shell 与脚本

Shell 是命令解释器。用户输入的 `ls`、`rm`、`cd` 等命令不能直接被内核识别，需要 Shell 解析后交给系统执行。

CentOS 7 常用解释器：

```bash
/bin/bash
```

脚本第一行通常写 shebang：

```bash
#!/bin/bash
```

示例脚本：

```bash
#!/bin/bash
echo '11111'
```

注释：

```bash
# 这是注释
```

脚本中建议保留日期、版本和必要说明，方便后续维护。

## Shell 脚本执行方式

### 赋予执行权限后执行

```bash
chmod a+x hello.sh
./hello.sh
/root/hello.sh
```

这种方式会根据 shebang 指定的解释器执行。

### 使用 bash 调用

```bash
bash hello.sh
sh hello.sh
```

这种方式不要求脚本有执行权限。

### 使用 source 执行

```bash
source hello.sh
. hello.sh
```

`source` 会在当前 Shell 中执行脚本。普通 `bash hello.sh` 会启动子 Shell，父进程无法直接获得子进程中的变量变化。

| 执行方式 | 是否需要执行权限 | 是否新建子 Shell | 是否影响当前 Shell 变量 |
| --- | --- | --- | --- |
| `./hello.sh` | 需要 | 是 | 否 |
| `bash hello.sh` | 不需要 | 是 | 否 |
| `source hello.sh` | 不需要 | 否 | 是 |

## crontab 中运行脚本

计划任务的环境变量很少，不能假设手工执行正常的脚本在 cron 中也正常。

建议：

1. 脚本里的命令使用绝对路径。
2. 脚本里的文件路径使用绝对路径。
3. 复杂任务先命令行测试，再脚本测试，再放进 `crontab`。
4. 必要时在脚本开头设置 `PATH`。

示例：

```bash
#!/bin/bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
/usr/bin/date >> /var/log/myjob.log
```

## 变量

### 变量赋值规则

```bash
name="cjk"
echo "$name"
```

规则：

| 规则 | 说明 |
| --- | --- |
| 变量名 | 可由字母、数字、下划线组成，不能以数字开头 |
| 等号 | `=` 两边不能有空格 |
| 值中有空格 | 必须用引号包起来 |
| 命令结果赋值 | 推荐使用 `$()` |
| 变量叠加 | 使用 `${变量名}` 更清晰 |

示例：

```bash
name="love cjk"
test="123"
test="${test}456"
now=$(date)
```

错误示例：

```bash
111name="cjk"
name = "bdyjy"
```

### 单引号与双引号

单引号不解析变量：

```bash
bn='nihao'
abc='hello,${bn}'
echo "${abc}"
```

输出：

```text
hello,${bn}
```

双引号会解析变量：

```bash
bn='nihao'
abc="hello,${bn}"
echo "${abc}"
```

输出：

```text
hello,nihao
```

建议默认使用双引号包裹变量，避免空格和通配符带来的问题：

```bash
echo "${name}"
```

### Bash 中的数字运算

Bash 变量默认按字符串处理，数学运算需要专门语法。

推荐使用：

```bash
num1=100
num2=200
num3=$((num1 + num2))
echo "${num3}"
```

也可以使用 `let`：

```bash
let num=1+20
echo "${num}"
```

`expr` 可用于简单运算，也常用于判断输入是否为整数：

```bash
expr 1 + 2
expr 1 + 'sss'
echo $?
```

注意：`expr` 计算结果为 `0` 时，`$?` 可能返回 `1`。判断整数时不能只按 `0/非0` 简单处理。

更稳的整数判断方式：

```bash
if [[ "$1" =~ ^-?[0-9]+$ ]]; then
    echo "integer"
else
    echo "not integer"
fi
```

## 环境变量

查看环境变量：

```bash
env
```

常见变量：

| 变量 | 含义 |
| --- | --- |
| `HOSTNAME` | 主机名 |
| `TERM` | 终端类型 |
| `SHELL` | 当前 Shell |
| `HISTSIZE` | 历史命令保留条数 |
| `SSH_CLIENT` | SSH 客户端 IP 和端口 |
| `USER` | 当前用户 |
| `PATH` | 命令查找路径 |
| `PWD` | 当前目录 |
| `LANG` | 语言环境 |
| `HOME` | 当前用户家目录 |
| `LOGNAME` | 登录用户名 |
| `_` | 上次执行命令的最后一个参数或命令本身 |

### PATH

```bash
echo "$PATH"
```

`PATH` 使用冒号分隔多个路径。系统执行命令时，会按顺序在这些路径中查找。

临时添加路径：

```bash
export PATH="/opt/bin:$PATH"
```

### LANG

```bash
echo "$LANG"
```

常见值：

```text
en_US.UTF-8
zh_CN.UTF-8
```

服务器环境通常更倾向使用英文，便于日志、报错和脚本处理。

## 特殊变量

| 变量 | 含义 |
| --- | --- |
| `$0` | 当前执行的脚本名，可能带路径 |
| `${1}` | 第一个参数 |
| `${n}` | 第 n 个参数 |
| `$#` | 参数总数 |
| `$@` | 所有参数，按独立参数展开 |
| `$?` | 上一个命令的退出状态 |
| `$$` | 当前 Shell 进程 PID |
| `$PPID` | 当前进程父进程 PID |

示例：

```bash
echo "$#"
echo "$0"
echo "$@"
echo "${3}"
```

执行：

```bash
sh aa.sh 11 22 wl home hello abc
```

输出含义：

| 输出 | 含义 |
| --- | --- |
| `6` | 参数总数 |
| `aa.sh` | 脚本名 |
| `11 22 wl home hello abc` | 所有参数 |
| `wl` | 第三个参数 |

### dirname 与 basename

```bash
dirname "$0"
basename "$0"
```

如果执行：

```bash
sh /root/bb.sh
```

结果：

```text
/root
bb.sh
```

## `$?` 与 exit

`$?` 获取上一个命令的退出状态：

| 返回值 | 含义 |
| --- | --- |
| `0` | 成功 |
| 非 `0` | 失败或异常 |

示例：

```bash
hg=$(pwd)
echo $?

kk=$(cat haha.txt)
echo $?
```

`exit` 用于退出当前脚本，并指定脚本最终状态：

```bash
exit 0
exit 1
exit 12
```

范围通常是 `0-255`。不要使用负数。

## shift

`shift` 会让位置参数左移一次。

示例：

```bash
if [ "${1}" == "-c" ]; then
    shift
fi
echo "${1}"
```

执行：

```bash
sh dd.sh -c nihao
```

输出：

```text
nihao
```

执行：

```bash
sh dd.sh 22 33 44
```

输出：

```text
22
```

`shift` 常用于解析命令行选项。

## read

`read` 用于接收用户输入。

```bash
read -p "please input only 2 args:" a b
echo "${a}"
echo "${b}"
```

如果输入超过两个字段，第二个变量会接收剩余所有内容。

更稳写法：

```bash
read -r -p "input: " value
```

`-r` 可以避免反斜杠被转义处理。

## 条件表达式

常见写法：

```bash
[ 条件表达式 ]
[[ 条件表达式 ]]
```

`[[ ]]` 是 Bash 扩展，功能更强，支持 `&&`、`||`、正则匹配等。

### 数字比较

在 `[ ]` 中使用：

| 操作符 | 含义 |
| --- | --- |
| `-eq` | 等于 |
| `-ne` | 不等于 |
| `-gt` | 大于 |
| `-ge` | 大于等于 |
| `-lt` | 小于 |
| `-le` | 小于等于 |

示例：

```bash
[ "$1" -gt 10 ] && echo "big"
```

不要在数字比较中随意使用 `>`、`<`，它们在不同上下文中可能按字符串比较或被 Shell 当成重定向符。

### 字符串比较

```bash
[ "come" == "hello" ]
[ "come" != "hello" ]
```

字符串变量建议加双引号：

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
| `!` | 取反 |

示例：

```bash
[ -f /root/aa.sh -a -f /root/bb.sh ] && echo 1 || echo 0
[[ -f /root/aa.sh && -f /root/bb.sh ]] && echo 1 || echo 0
```

## if

基础格式：

```bash
if [ 条件表达式 ]; then
    代码
fi
```

带 `else`：

```bash
if [ 条件表达式 ]; then
    代码
else
    代码
fi
```

多分支：

```bash
if [ 条件表达式1 ]; then
    代码
elif [ 条件表达式2 ]; then
    代码
else
    代码
fi
```

嵌套：

```bash
if [ 条件表达式1 ]; then
    if [ 条件表达式2 ]; then
        代码
    fi
fi
```

参数数量判断：

```bash
if [ $# -ne 2 ]; then
    echo "please input two args"
else
    echo "$1"
    echo "$2"
fi
```

判断一个参数，并显示输入值：

```bash
#!/bin/bash
if [ $# -ne 2 ]; then
    echo "please input two args"
    if [ $# -eq 1 ]; then
        echo "you input ${1}"
    fi
else
    echo "$1"
    echo "$2"
fi
```

## case

`case` 适合做固定值匹配。

```bash
case "变量" in
    值1)
        代码
        ;;
    值2)
        代码
        ;;
    *)
        默认代码
        ;;
esac
```

示例：

```bash
read -p "please input:" age
case "${age}" in
    10)
        echo "you input age is 10"
        ;;
    30)
        echo "you input age is 30"
        ;;
    [4-8])
        echo "you input age is ${age}"
        ;;
    *)
        echo "age error"
        ;;
esac
```

注意：`case` 中的 `[4-8]` 是单字符范围，只适合匹配一个字符，不能表示 `10-20` 这样的数值范围。

## while

语法：

```bash
while 条件表达式
do
    代码
done
```

无限循环：

```bash
while true
do
    uptime
    sleep 2
done
```

倒计时示例：

```bash
if [ $# -ne 1 ]; then
    echo "only one arg"
    exit 10
fi

expr 1 + "${1}" >/dev/null 2>&1
if [ $? -ne 0 -a $? -ne 1 ]; then
    echo "input int"
    exit 20
fi

i=${1}
while [ "${i}" -gt 0 ]
do
    echo "${i}"
    ((i--))
done
```

## while 读取文件

### exec 重定向

```bash
exec <文件名
while read line
do
    echo "${line}"
done
```

脚本示例：

```bash
#!/bin/bash
if [ $# -ne 1 ]; then
    echo "please input one args"
    exit 10
fi

if [ -f "$1" ]; then
    exec <"$1"
    while read line
    do
        echo "${line}"
    done
else
    echo "please input file"
fi
```

匹配包含 `sleep` 的行：

```bash
#!/bin/bash
if [ $# -ne 1 ]; then
    echo "please input one args"
    exit 10
fi

if [ -f "$1" ]; then
    exec <"$1"
    while read line
    do
        echo "${line}" | grep 'sleep' >/dev/null
        if [ $? -eq 0 ]; then
            echo "${line}"
        fi
    done
else
    echo "please input file"
fi
```

### 管道方式

```bash
cat 文件名 | while read line
do
    echo "${line}"
done
```

这种方式中 `while` 通常运行在子 Shell 中，循环内修改变量时容易遇到作用域问题。

### 输入重定向方式

```bash
while read line
do
    echo "${line}"
done < 文件名
```

更推荐写法：

```bash
while IFS= read -r line
do
    echo "${line}"
done < "$1"
```

## break 与 continue

| 关键字 | 作用 |
| --- | --- |
| `break` | 直接跳出循环 |
| `continue` | 跳过本次循环剩余代码，进入下一轮 |

示例：

```bash
age=$1
total=10

while [ "${total}" -gt 0 ]
do
    let total=${total}-1
    if [ "${age}" -lt 10 ]; then
        echo "${age}"
        let age=${age}-1
        if [ "${age}" -lt 8 ]; then
            break
        fi
    fi
    echo "hello"
done
```

## for

遍历列表：

```bash
for 变量名 in 变量取值列表
do
    代码
done
```

C 风格循环：

```bash
for ((表达式1; 表达式2; 表达式3))
do
    代码
done
```

示例：

```bash
for x in 22 "abc" 66 "nihao"
do
    echo "${x}"
done
```

C 风格示例：

```bash
for ((i=0; i<4; i++))
do
    echo "${i}"
done
```

### 用 for 批量改名

原始思路：

```bash
for x in $(ls | grep "txt$")
do
    mv "${x}" "$(echo "${x}" | cut -d . -f1).sh"
done
```

也可以用 `awk`：

```bash
for x in $(ls | grep "txt$")
do
    mv "${x}" "$(echo "${x}" | awk -F "." '{print $1}').sh"
done
```

更简单的方式：

```bash
rename "txt" "sh" *.txt
```

注意：`for x in $(ls)` 无法安全处理包含空格、换行符的文件名。更稳写法：

```bash
for x in *.txt
do
    [ -e "$x" ] || continue
    mv -- "$x" "${x%.txt}.sh"
done
```

## Shell 中的真值

在 Shell 中，非空字符串为真。`0` 是非空字符串，所以为真：

```bash
if [ 0 ]; then
    echo "aa"
else
    echo "b"
fi
```

输出：

```text
aa
```

数字 0 是否表示成功，是命令退出状态中的语义，不等同于字符串判断。

## 脚本编写建议

| 建议 | 说明 |
| --- | --- |
| 使用 `#!/bin/bash` | 明确解释器 |
| 变量加双引号 | 避免空格、通配符问题 |
| 使用 `${var}` | 变量边界更清楚 |
| 复杂命令先手工测试 | 再写进脚本 |
| cron 脚本写绝对路径 | 避免环境变量缺失 |
| 参数先校验 | 再执行危险操作 |
| 批量文件操作先预览 | 防止误改 |
| 必要时加调试选项 | `bash -x script.sh` |
