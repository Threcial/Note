---
type: Reference
status: Organized
source: day17.md
created: 2026-06-14
updated: 2026-06-14
Belongs to: "[[Nginx]]"
Related to:
  - "[[Nginx 配置文件]]"
  - "[[正则表达式]]"
Has:
  - "[[rewrite]]"
  - "[[Nginx set]]"
  - "[[Nginx if]]"
  - "[[return]]"
---
# Nginx URL 重写、变量与 if 判断

Nginx 的 URL 重写主要由 `rewrite` 指令完成，本质是根据正则匹配 URI，再把请求改写到新的 URI 或直接让浏览器跳转到新 URL。是否改变浏览器地址栏、是否重新进入 location 匹配流程，取决于 `flag`。

`rewrite` 不应被当作万能跳转工具。简单跳转优先考虑 `return 301/302`；复杂 URI 改写再使用 `rewrite`。这样配置更直观，也更少踩 `last`、`break` 的坑。

## 总览

| 条目          | 关系                      |
| ----------- | ----------------------- |
| `rewrite`   | 根据正则改写 URI 或返回重定向       |
| `permanent` | 返回 301，浏览器地址栏变化         |
| `redirect`  | 返回 302，浏览器地址栏变化         |
| `last`      | 重新发起一次 location 匹配      |
| `break`     | 停止当前 rewrite 链，不再继续向后匹配 |
| `set`       | 定义 Nginx 变量，值是字符串       |
| `if`        | 条件判断，常与 `return` 搭配     |
| `return`    | 直接返回状态码或跳转 URL          |

## rewrite 基本语法

```nginx
rewrite 正则 替换内容 flag;
```

示例：

```nginx
location / {
    rewrite ^/old/(.*)$ /new/$1 permanent;
}
```

含义：

| 部分            | 含义           |
| ------------- | ------------ |
| `rewrite`     | 指令名          |
| `^/old/(.*)$` | 要匹配的 URI     |
| `/new/$1`     | 替换后的 URI     |
| `permanent`   | 返回 301 永久重定向 |

## flag 的区别

| flag        | 状态码       | 浏览器地址栏 | 后续行为             |
| ----------- | --------- | ------ | ---------------- |
| `permanent` | 301       | 改变     | 浏览器重新请求新 URL     |
| `redirect`  | 302       | 改变     | 浏览器临时跳转          |
| `last`      | 200 或后续结果 | 通常不变   | 重新进入 location 匹配 |
| `break`     | 200 或后续结果 | 通常不变   | 停止当前 rewrite 链   |

### permanent

适合旧域名永久迁移到新域名。

```nginx
location / {
    rewrite ^/(.*)$ https://www.example.com/$1 permanent;
}
```

更推荐的简单写法：

```nginx
server {
    listen 80;
    server_name old.example.com;
    return 301 https://new.example.com$request_uri;
}
```

### redirect

适合临时跳转。

```nginx
location /sg {
    rewrite ^/sg$ https://www.baidu.com redirect;
}
```

### last

`last` 会重新进入 location 匹配流程。未显式写 flag 时，rewrite 默认行为接近 `last`。

```nginx
location / {
    rewrite ty$ /sanguo11 last;
    root html;
}

location /sanguo11 {
    rewrite 11$ /2222.html last;
    root html;
}
```

如果请求命中第一条规则，会继续匹配 `/sanguo11`，然后再改写到 `/2222.html`。

### break

`break` 停止当前 rewrite 处理，不重新发起新的 location 匹配。

```nginx
location / {
    rewrite ty$ /sanguo11 break;
    root html;
}
```

此时 Nginx 会尝试在当前 location 的 `root` 规则下寻找 `/sanguo11` 对应的文件，而不会继续匹配 `location /sanguo11`。

## 目录重写的斜杠问题

如果重写目标是目录，建议结尾加 `/`。

```nginx
location /sanguo22 {
    rewrite 22$ /sanguo33/ last;
    root html;
    index index.html;
}
```

目录不加 `/` 容易触发额外的重定向或导致 location 匹配行为和预期不一致。`break` 场景下也可能因为目录补斜杠导致行为看起来像失效。

## Nginx 正则与 location 符号

| 符号   | 含义         |
| ---- | ---------- |
| `~`  | 区分大小写正则匹配  |
| `~*` | 不区分大小写正则匹配 |
| `!~` | 正则不匹配      |
| `=`  | 精确匹配       |
| `^`  | 开始位置       |
| `$`  | 结束位置       |
| `.`  | 匹配任意单个字符   |
| `*`  | 重复 0 次或多次  |

`location =` 更适合匹配具体文件或精确 URI，不适合当作目录匹配规则。

## set 定义变量

Nginx 变量是字符串值。

```nginx
server {
    location / {
        set $target_path 1111;
        rewrite ^(.*)$ http://192.168.111.12/$target_path/ permanent;
    }
}
```

变量作用域取决于定义位置：

| 定义位置       | 可见范围                  |
| ---------- | --------------------- |
| `server`   | 当前 server 内的 location |
| `location` | 当前 location 内         |

## if 与 return

根据 User-Agent 拦截 elinks：

```nginx
server {
    location / {
        if ($http_user_agent ~* 'elinks') {
            return 404;
        }

        root html;
        index index.html;
    }
}
```

`if` 在 Nginx 中容易被滥用。安全使用方式主要是配合 `return` 或 `rewrite ... last` 这类简单逻辑。复杂逻辑应尽量使用 `map`、独立 `location` 或后端服务处理。

## break 终止后续指令

```nginx
server {
    location / {
        if ($http_user_agent ~* 'elinks') {
            break;
            return 404;
        }

        root html;
        index index.html;
    }
}
```

`break` 后面的 `return 404` 不会执行，所以最终可能仍然返回正常页面。这类配置容易误判，不建议把 `break` 和 `return` 混在一起写。

## 现代写法补充：return 优先于简单 rewrite

域名跳转：

```nginx
server {
    listen 80;
    server_name old.example.com;
    return 301 https://new.example.com$request_uri;
}
```

HTTP 跳 HTTPS：

```nginx
server {
    listen 80;
    server_name www.example.com;
    return 301 https://$host$request_uri;
}
```

固定 URI 跳转：

```nginx
location = /old.html {
    return 301 /new.html;
}
```

这些场景用 `return` 比 `rewrite` 更清晰。

## 命令速查

| 目的       | 命令                                  |
| -------- | ----------------------------------- |
| 检查配置     | `nginx -t`                          |
| 查看完整配置   | `nginx -T`                          |
| 重载配置     | `systemctl reload nginx`            |
| 测试重定向状态码 | `curl -I http://example.com/old`    |
| 跟随跳转     | `curl -IL http://example.com/old`   |
| 查看访问日志   | `tail -f /var/log/nginx/access.log` |
| 查看错误日志   | `tail -f /var/log/nginx/error.log`  |

## 易错点

| 问题                   | 原因                                |
| -------------------- | --------------------------------- |
| 写了 rewrite 但浏览器地址栏没变 | 使用的是 `last` / `break`，不是 301/302  |
| 跳转缓存一直存在             | 浏览器缓存了 301                        |
| 目录跳转异常               | 目标目录缺少结尾 `/`                      |
| `if` 逻辑不生效           | `break` 提前终止，或 if 使用位置不合理         |
| 复杂 rewrite 难排查       | 使用 `nginx -T` 和 `curl -I/-L` 分步验证 |

## 关联条目

| 类型 | 条目                                               |
| -- | ------------------------------------------------ |
| 前置 | [[Nginx 配置文件]]、[[Nginx location]]、[[正则表达式]]      |
| 核心 | [[rewrite]]、[[return]]、[[HTTP 301]]、[[HTTP 302]] |
| 延伸 | [[Nginx 反向代理]]、[[Nginx 虚拟主机]]、[[SEO 与重定向]]       |
