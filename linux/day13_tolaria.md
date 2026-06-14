---
title: Shell 进阶：select、数组、函数、Expect、并发与随机数
type: knowledge
status: evergreen
source: day13.md
created: 2026-06-13
updated: 2026-06-13
tags:
  - Linux
  - Shell
  - Bash
  - select
  - 数组
  - 函数
  - Expect
  - 并发
  - 随机数
aliases:
  - DAY13
  - Shell 进阶
  - Shell 函数
  - Expect 自动交互
Belongs to:
  - "[[Shell 编程]]"
  - "[[Linux 运维自动化]]"
Related to:
  - "[[DAY11 - Shell 脚本基础、变量、条件判断与循环]]"
  - "[[SSH 密钥登录]]"
  - "[[rsync]]"
  - "[[grep]]"
  - "[[awk]]"
  - "[[Shell 重定向]]"
Has:
  - "[[select]]"
  - "[[Shell 数组]]"
  - "[[参数展开]]"
  - "[[Shell 函数]]"
  - "[[Expect]]"
  - "[[Shell 调试]]"
  - "[[Shell 并发]]"
  - "[[wait]]"
  - "[[RANDOM]]"
  - "[[uuidgen]]"
  - "[[md5sum]]"
  - "[[cut]]"
_organized: true
---
# Shell 进阶：select、数组、函数、Expect、并发与随机数

## 概览

|          |                                               |
| -------- | --------------------------------------------- |
| `select` | 适合写简单交互菜单，通常配合 `case` 使用                      |
| 数组       | Bash 只支持一维数组，常用于批量参数、菜单、列表处理                  |
| 参数展开     | `${var##*/}`、`${var%/*}` 等适合做路径和后缀处理          |
| 函数       | 函数用于封装逻辑，`return` 返回状态码，`exit` 退出整个脚本         |
| Expect   | 用于处理交互式命令，常见于 ssh-keygen、ssh-copy-id、passwd 等 |
| 并发       | `&` + `wait` 可以让循环中的任务并发执行                    |
| 随机数      | `$RANDOM`、`date +%s%N                         |

## select 菜单

`select` 用于生成交互式菜单。

语法：

```bash
select 变量名 in 菜单取值列表
do
    代码
done
```

示例：

```bash
select name in haha hello welcome
do
    echo "${name}"
done
```

执行后显示：

```text
1) haha
2) hello
3) welcome
#?
```

用户输入数字后，变量 `name` 会得到对应菜单项。

## PS3 与 REPLY

默认提示符 `#?` 不直观，可以设置 `PS3`。

```bash
PS3="please input:"
select name in haha hello welcome
do
    echo "${name}"
    echo "${REPLY}"
done
```

| 变量      | 含义            |
| ------- | ------------- |
| `PS3`   | `select` 的提示符 |
| `REPLY` | 用户输入的原始编号     |

## select 配合 case

```bash
PS3="please input:"
select name in first printfile printdir quit
do
    case "${REPLY}" in
        1)
            echo "your choose is the first"
            ;;
        2)
            ls
            ;;
        3)
            pwd
            ;;
        4)
            exit
            ;;
        *)
            echo "input error"
            ;;
    esac
done
```

`select` 负责展示菜单，`case` 负责根据用户选择执行不同逻辑。

## 嵌套 select

嵌套菜单中，内层 `select` 需要使用 `break` 退出当前层级。

```bash
PS3="please input:"
select name in first printfile printdir quit
do
    case "${REPLY}" in
        1)
            echo "your choose is the first"
            select sub in second printfile printdir quit
            do
                case "${REPLY}" in
                    1)
                        echo "your choose is the second"
                        ;;
                    2)
                        ls
                        ;;
                    3)
                        pwd
                        ;;
                    4)
                        break
                        ;;
                    *)
                        echo "input error"
                        ;;
                esac
            done
            ;;
        2)
            ls
            ;;
        3)
            pwd
            ;;
        4)
            exit
            ;;
        *)
            echo "input error"
            ;;
    esac
done
```

## Shell 数组

Bash 数组只支持一维数组。

定义：

```bash
aa=(1 2 3)
```

输出所有元素：

```bash
echo "${aa[@]}"
echo "${aa[*]}"
```

推荐使用：

```bash
"${aa[@]}"
```

获取数组长度：

```bash
echo "${#aa[@]}"
```

取单个元素：

```bash
echo "${aa[0]}"
echo "${aa[1]}"
```

数组下标从 0 开始。

### 遍历数组

```bash
aa=(1 2 3 5)

for ((i=0; i<${#aa[@]}; i++))
do
    echo "${aa[i]}"
done
```

更常用写法：

```bash
for item in "${aa[@]}"
do
    echo "${item}"
done
```

### 命令结果转数组

```bash
aa=($(ls))
for ((i=0; i<${#aa[@]}; i++))
do
    echo "${aa[i]}"
done
```

这种写法无法安全处理带空格的文件名。更稳的文件遍历应使用通配符、`find -print0` 或 `readarray`。

### 数组校验用户输入

结合 `select`：

```bash
PS3="please input:"
select name in dayin xianshiwj xianshimulu tuichu
do
    gy=(${REPLY})
    if [ "${#gy[@]}" -ne 1 ]; then
        echo "only one arg"
        exit 1
    fi
done
```

结合 `read -p`：

```bash
read -p "please:" a
name=(${a})
if [ "${#name[@]}" -ne 1 ]; then
    echo "only one arg"
fi
```

如果只是判断用户输入了几个字段，数组可以把输入按空格切开再统计。

## 参数展开

参数展开适合处理路径、文件名、后缀。

| 表达式           | 含义                              |
| ------------- | ------------------------------- |
| `${file#*/}`  | 删除第一个 `/` 及其左侧内容                |
| `${file##*/}` | 删除最后一个 `/` 及其左侧内容，相当于取 basename |
| `${file#*.}`  | 删除第一个 `.` 及其左侧内容                |
| `${file##*.}` | 删除最后一个 `.` 及其左侧内容，相当于取扩展名       |
| `${file%%/*}` | 删除第一个 `/` 及其右侧内容                |
| `${file%/*}`  | 删除最后一个 `/` 及其右侧内容，相当于取 dirname  |
| `${file%%.*}` | 删除第一个 `.` 及其右侧内容                |
| `${file%.*}`  | 删除最后一个 `.` 及其右侧内容               |

示例：取文件名。

```bash
aa=${1##*/}
echo "${aa}"
```

执行：

```bash
sh bb.sh /root/hah/www/kkk
```

输出：

```text
kkk
```

示例：取目录名。

```bash
aa=${1%/*}
echo "${aa}"
```

执行：

```bash
sh bb.sh /root/hah/www/kkk
```

输出：

```text
/root/hah/www
```

与外部命令对比：

```bash
basename /root/hah/www/kkk
dirname /root/hah/www/kkk
```

参数展开不启动外部进程，脚本中大量处理字符串时效率更好。

## Shell 函数

函数写法：

```bash
function 函数名() {
    代码
    return n
}
```

也可以省略 `function`：

```bash
函数名() {
    代码
    return n
}
```

函数调用只写函数名，不加括号：

```bash
函数名
```

命名建议：

| 对象 | 建议                                       |
| -- | ---------------------------------------- |
| 变量 | 全小写，下划线连接，例如 `local_plus_script`         |
| 函数 | 首字母大写或单词首字母大写，下划线连接，例如 `Local_Plus_Name` |

## return 与 exit

| 关键字      | 作用          |
| -------- | ----------- |
| `return` | 从函数返回，返回状态码 |
| `exit`   | 退出整个脚本      |

示例：

```bash
#!/bin/bash

Echo_Name() {
    echo "hello"
    return 100
}

Echo_Name
echo $?
```

如果函数中不写 `return`，默认返回最后一条命令的退出状态。

当函数执行后需要立即终止整个脚本时，应使用 `exit`。

## 函数参数

函数接收参数的方式与脚本相同：

| 变量     | 含义             |
| ------ | -------------- |
| `${1}` | 第一个函数参数        |
| `${2}` | 第二个函数参数        |
| `$#`   | 函数参数数量         |
| `$@`   | 函数所有参数         |
| `$0`   | 仍表示调用脚本名，不是函数名 |

示例：

```bash
haha() {
    echo "nihao"
    echo "${1}"
}

nihao() {
    echo "welcome"
    echo "${2}"
    echo "$#"
}

haha /root/aaa
nihao /ppp kkk
```

输出：

```text
nihao
/root/aaa
welcome
kkk
2
```

把脚本参数传给函数：

```bash
haha "$@"
nihao "$@"
```

推荐写成 `"$@"`，这样可以保留每个参数的边界。

## 函数封装参数判断

```bash
#!/bin/bash

Hello() {
    echo "${0} only one args"
    exit 1
}

Only() {
    echo "you input is ${1}"
}

Main() {
    if [ $# -ne 1 ]; then
        Hello
    fi
    Only "${1}"
}

Main "$@"
```

输入多个参数：

```bash
sh cc.sh sssss aaaa
```

输出：

```text
cc.sh only one args
```

输入一个参数：

```bash
sh cc.sh sssss
```

输出：

```text
you input is sssss
```

## 使用系统函数库显示 OK/FAILED

CentOS 中可以导入系统函数库：

```bash
source /etc/init.d/functions
```

示例：

```bash
source /etc/init.d/functions

if [ $# -ne 1 ]; then
    action "only one args" /bin/false
    exit 1
fi

action "you input is ${1}" /bin/true
```

| 命令           | 显示效果       |
| ------------ | ---------- |
| `/bin/true`  | `[ OK ]`   |
| `/bin/false` | `[FAILED]` |

这种方式适合 CentOS 系统脚本，不一定适合所有发行版。

## local 局部变量

函数内变量默认也是全局变量。定义局部变量应使用 `local`：

```bash
Demo() {
    local name="inside"
    echo "${name}"
}
```

局部变量在函数结束后失效。

## Expect 自动交互

`expect` 用于处理需要交互输入的命令，例如 `ssh-keygen`、`ssh-copy-id`、`passwd`。

安装：

```bash
yum install expect -y
```

基本脚本：

```expect
#!/usr/bin/expect
set timeout -1

spawn ssh-keygen
expect "id_rsa):"
send "\r"
expect "passphrase):"
send "\r"
expect "again:"
send "\r"
expect eof

spawn ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.31.181
expect "(yes/no)?"
send "yes\r"
expect "password"
send "111111\r"
expect eof
```

执行：

```bash
expect script.exp
```

## Expect 分支匹配

```expect
#!/usr/bin/expect
set timeout 10
spawn ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.111.10

expect {
    "(yes/no)?" {
        send "yes\r"
        expect "password"
        send "111111\r"
    }
    "password" {
        send "111111\r"
    }
    "WARNING" {
        exit 0
    }
    timeout {
        exit 101
    }
}

expect eof
```

`timeout` 可用于处理远端无响应，返回特定状态码，便于 Shell 判断。

## Shell 调用 Expect

批量场景常用 Shell 循环包裹 Expect。

```bash
#!/bin/bash

for x in 192.168.31.181 192.168.31.182
do
expect << eof
set timeout 10
spawn ssh-copy-id -i /root/.ssh/id_rsa.pub root@${x}
expect "(yes/no)?"
send "yes\r"
expect "password"
send "111111\r"
expect eof
eof
done
```

注意：这里的结束标记 `eof` 前后不能有空格，否则 heredoc 无法正确结束。

原文中说 “expect 不支持循环” 不严谨。Expect 基于 Tcl，实际支持控制结构；但在运维批量任务里，用 Shell 控制循环、Expect 处理交互通常更直观。

## Shell 调试

| 参数                | 作用           |
| ----------------- | ------------ |
| `sh -n script.sh` | 只检查语法，不执行    |
| `sh -v script.sh` | 先打印脚本内容，再执行  |
| `sh -x script.sh` | 执行并显示每一步展开结果 |

常用：

```bash
bash -x script.sh
```

也可以在脚本内局部开启调试：

```bash
set -x
命令
set +x
```

## if 多条件判断

```bash
if [ "${1}" -eq 100 ] && [ "${2}" -eq 200 ]; then
    echo ok
fi

if [ "${1}" -eq 100 ] || [ "${2}" -ne 20 ]; then
    echo ok
fi
```

使用 [[ ]] 更适合复杂条件：

```bash
if [[ "${1}" -eq 100 && "${2}" -eq 200 ]]; then
    echo ok
fi
```

## if 中使用正则

原始方式：

```bash
if echo "${1}" | grep "^a" >/dev/null
then
    echo "match"
fi
```

更推荐 Bash 原生写法：

```bash
if [[ "${1}" =~ ^a ]]; then
    echo "match"
fi
```

区别：

| 写法       | 特点                    |
| -------- | --------------------- |
| `echo    | grep`                 |
| [[ =~ ]] | Bash 内建，效率更高，适合脚本内部判断 |

## Shell 并发

在命令后加 `&` 可以让任务进入后台，并发执行。使用 `wait` 等待所有后台任务结束。

```bash
#!/bin/bash
for i in {1..254}
do
    ping -c 10 192.168.31.${i} &
done
wait
```

多行代码并发执行，需要用 `{}` 包起来：

```bash
#!/bin/bash

for ((i=1; i<255; i++))
do
{
    alive=$(ping -c 5 192.168.111.${i} | grep received | awk '{print $4}')
    if [ "${alive}" -eq 0 ]; then
        echo 192.168.111.${i} >>/tmp/tmp-error.txt
    else
        echo 192.168.111.${i} >>/tmp/tmp-alive.txt
    fi
} &
done

wait
sort -t '.' -n -k 4 /tmp/tmp-error.txt > /tmp/error.txt
sort -t '.' -n -k 4 /tmp/tmp-alive.txt > /tmp/alive.txt
echo "finish"
```

并发能提高网络和磁盘批量任务效率，但会增加 CPU、内存、文件描述符和网络压力。生产脚本建议控制并发数量。

简单限流示例：

```bash
max_jobs=20

for i in {1..254}
do
{
    ping -c 2 192.168.31.${i} >/dev/null && echo "192.168.31.${i} alive"
} &

    while [ "$(jobs -rp | wc -l)" -ge "${max_jobs}" ]
    do
        sleep 0.2
    done
done

wait
```

## 随机数生成

### RANDOM

```bash
echo "$RANDOM"
```

`$RANDOM` 生成 `0-32767` 之间的随机数。

### date + md5sum

```bash
date +%s%N | md5sum
```

`%s` 是秒，`%N` 是纳秒；再配合 `md5sum` 可以得到看起来更随机的字符串。

截取部分字符：

```bash
date +%s%N | md5sum | cut -c 1-8
```

### uuidgen

```bash
uuidgen
```

`uuidgen` 适合生成唯一标识。用于网卡 UUID、临时文件名、任务 ID 等场景比较合适。

### cut 截取字符

```bash
cut -c 2-5 11.txt
cut -c 1,3,6 11.txt
```

| 参数         | 含义                 |
| ---------- | ------------------ |
| `-c 2-5`   | 取每行第 2 到第 5 个字符    |
| `-c 1,3,6` | 取每行第 1、第 3、第 6 个字符 |

## 易错点

| 问题                    | 说明                           |
| --------------------- | ---------------------------- |
| `select` 输入非法值        | 需要用 `case` 的 `*` 分支处理        |
| 数组展开不加引号              | 参数边界可能被空格打散                  |
| 函数里用 `return` 想退出脚本   | `return` 只退出函数，退出脚本要用 `exit` |
| Expect heredoc 结束符有空格 | 会导致 Shell 找不到结束位置            |
| 并发任务不 `wait`          | 主脚本可能提前结束                    |
| 并发数量不限制               | 可能导致负载、文件描述符或网络压力过高          |
| `$RANDOM` 范围有限        | 不适合需要全局唯一的场景                 |
