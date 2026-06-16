---
type: knowledge
status: evergreen
created: 2026-06-13
updated: 2026-06-13
related_to:
  - "[[shell-脚本基础]]"
  - "[[Linux 运维自动化]]"
has:
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
  - "[[cut]]"
_organized: true
---

# Shell 脚本进阶

## select 菜单

```bash
select name in haha hello welcome
do
    echo "${name}"
done
```

执行后显示编号菜单，用户输入数字得到对应项。

| 变量 | 含义 |
| --- | --- |
| `PS3` | `select` 的提示符 |
| `REPLY` | 用户输入的原始编号 |

### select 配合 case

```bash
PS3="please input:"
select name in first printfile quit
do
    case "${REPLY}" in
        1) echo "your choose is the first" ;;
        2) ls ;;
        3) exit ;;
        *) echo "input error" ;;
    esac
done
```

## Shell 数组

Bash 只支持一维数组。

```bash
aa=(1 2 3)
echo "${aa[@]}"      # 所有元素
echo "${#aa[@]}"     # 数组长度
echo "${aa[0]}"      # 第一个元素
```

遍历数组：

```bash
for item in "${aa[@]}"; do
    echo "${item}"
done
```

## 参数展开

| 表达式 | 含义 |
| --- | --- |
| `${file##*/}` | 删除最后一个 `/` 及其左侧，相当于 basename |
| `${file%/*}` | 删除最后一个 `/` 及其右侧，相当于 dirname |
| `${file##*.}` | 取扩展名 |
| `${file%.*}` | 删除最后一个 `.` 及其右侧 |

参数展开不启动外部进程，脚本中大量处理字符串时效率更好。

## Shell 函数

```bash
函数名() {
    代码
    return n
}
```

调用：`函数名`

### return 与 exit

| 关键字 | 作用 |
| --- | --- |
| `return` | 从函数返回，返回状态码 |
| `exit` | 退出整个脚本 |

### 函数参数

```bash
haha() {
    echo "${1}"
}
haha "$@"     # 把脚本参数传给函数
```

### local 局部变量

```bash
Demo() {
    local name="inside"
    echo "${name}"
}
```

### 系统函数库

CentOS 中：

```bash
source /etc/init.d/functions
action "message" /bin/true    # 显示 [ OK ]
action "message" /bin/false   # 显示 [FAILED]
```

## Expect 自动交互

用 Expect 处理交互式命令：

```expect
#!/usr/bin/expect
set timeout -1
spawn ssh-copy-id -i /root/.ssh/id_rsa.pub root@192.168.31.181
expect "(yes/no)?"
send "yes\r"
expect "password"
send "111111\r"
expect eof
```

Shell 循环包裹 Expect：

```bash
for x in 192.168.31.181 192.168.31.182
do
expect << eof
set timeout 10
spawn ssh-copy-id -i /root/.ssh/id_rsa.pub root@${x}
expect "(yes/no)?" { send "yes\r"; expect "password"; send "111111\r" }
expect eof
eof
done
```

## Shell 调试

```bash
bash -x script.sh
```

脚本内局部开启调试：

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

if [[ "${1}" =~ ^a ]]; then
    echo "match"
fi
```

## Shell 并发

```bash
for i in {1..254}
do
    ping -c 10 192.168.31.${i} &
done
wait
```

限制并发数：

```bash
max_jobs=20
for i in {1..254}
do
    {
        ping -c 2 192.168.31.${i} >/dev/null && echo "192.168.31.${i} alive"
    } &
    while [ "$(jobs -rp | wc -l)" -ge "${max_jobs}" ]; do
        sleep 0.2
    done
done
wait
```

## 随机数生成

```bash
echo "$RANDOM"                      # 0-32767
date +%s%N | md5sum | cut -c 1-8   # 随机字符串
uuidgen                              # 唯一标识
```

## cut 截取字符

```bash
cut -c 2-5 11.txt
cut -c 1,3,6 11.txt
```

## 易错点

| 问题 | 说明 |
| --- | --- |
| `select` 输入非法值 | 需要用 `case` 的 `*` 分支处理 |
| 数组展开不加引号 | 参数边界可能被空格打散 |
| 函数里用 `return` 想退出脚本 | `return` 只退出函数 |
| 并发任务不 `wait` | 主脚本可能提前结束 |
| 并发数量不限制 | 可能导致负载过高 |
| `$RANDOM` 范围有限 | 不适合需要全局唯一的场景 |
