---
type: Practice
has:
  - "[[linux/linux-安装与初始化]]"
  - "[[linux/linux-基础命令]]"
related_to:
  - "[[web中间件/nginx-优化与负载均衡]]"
  - "[[web中间件/nginx-反向代理与限速]]"
  - "[[web中间件/nginx-安装与配置结构]]"
  - "[[web中间件/nginx-虚拟主机与访问控制]]"
---
# 网站性能优化记录

> 服务器：Linux ，Nginx，PHP ，WordPress ，Sakurairo 主题\
> 硬件：轻量应用服务器

***

## 1. linux 优化

1. 云服务器未配置 swap ，手动增加并配置 swap 使用优先级
2. 开启防火墙只放行必要端口
3. 修改常用端口
4. 针对 ssh 远程连接，配置密钥登录，关闭密码登录和无密码登录
5. 安装 fail2ban 配置规则自动封禁异常 ip
6. 配置定时任务监控服务器状态，异常报警

## 2. nginx 优化

1. 配置 ssl
2. nginx 层面禁止访问敏感路径

## 3. php 优化

1. 配置参数优化，减小内存占用

## 4. wordpress 优化

1. 修改登录页地址
2. 创建页面缓存，提高访问速度
3. 创建本地字体库

4.

***

## 阶段一：诊断分析

### 测速方法

逐资源 `curl -o /dev/null -s -w` 测速，另加 `-H "User-Agent: ...iPhone..."` 测试移动端。

### 核心发现

**从本 VM（海外）测试：**

| 资源                                           | 耗时          | 大小    | 托管位置                    |
| -------------------------------------------- | ----------- | ----- | ----------------------- |
| 首页 HTML                                      | 2.70s       | 46KB  | 源站服务器                   |
| 主 CSS（PHP 动态 `?sakura_header&github&minify`） | **12.77s**  | 168KB | 源站服务器（`index.php` 动态输出） |
| app.js                                       | 3.67s       | 69KB  | 源站服务器                   |
| nav.js                                       | 3.39s       | 55KB  | 源站服务器                   |
| Font Awesome CSS                             | **0.13s** ✅ | 74KB  | CDN（s4.zstatic.net）     |
| 首图                                           | **0.31s** ✅ | 501KB | CDN（yppp.net）           |

注意：12.77s 是 **从本 VM 到源站的跨国网络 + PHP 执行** 的叠加耗时，不代表国内用户访问速度。

**从服务器本地测试：**

| 资源          | 耗时    |
| ----------- | ----- |
| 首页 HTML     | 0.09s |
| 动态 CSS      | 0.09s |
| 静态 CSS（如存在） | 0.02s |

**说明：** 12.77s 与 0.09s 的差异主要是 **网络延迟 + 带宽瓶颈** 而非 PHP 动态生成的开销。后文的优化效果评估应以同一测试口径为基准。

***

## 阶段二：CSS 静态化

### 背景

Sakurairo 主题的 `css/index.php` 会根据 GET 参数（`sakura_header`、`content_style`、`wave`、`minify`、版本号）动态读取并合并多个 CSS 文件输出。

### 静态化方案

```bash
# 注意：以下命令未覆盖所有情况，详见「不严谨之处」
cd /usr/share/nginx/html/wp-content/themes/Sakurairo/css/
php -r '
$_GET["sakura_header"] = "1";
$_GET["github"] = "1";
$_GET["no_wave"] = "1";
$_GET["minify"] = "1";
$_GET["3"] = "10";
require "index.php";
' > /tmp/sakurairo-static.css
chown nginx:nginx /tmp/sakurairo-static.css
cp /tmp/sakurairo-static.css /usr/share/nginx/html/wp-content/themes/Sakurairo/css/sakurairo-static.css
```

### 修改方法（直接改主题文件，非最佳实践）

`functions.php` 第 523 行：

```php
// 改前：动态 URL，每次请求 PHP 重新编译
$iro_css = $core_lib_basepath . '/css/' . $index . '?' . $sakura_header . '&' . $content_style . '&' . $wave . '&minify&' . IRO_VERSION;

// 改后：指向静态文件
$iro_css = $core_lib_basepath . '/css/sakurairo-static.css';
```

### 效果

- 服务器本地 CSS 耗时：**0.085s → 0.019s**
- 从本 VM 远端测：受网络瓶颈限制，差异不明显（均在 1.5-3s 左右）

### ⚠️ 不严谨之处

1. **PHP 生成命令没有处理** `$index` **变量。** 原始代码 `$index` 是空字符串或 `'index.php'`，取决于固定链接设置（`permalink_structure` 是否包含 `index.php`）。硬编码跳过此逻辑可能导致生成路径与主题实际 URL 不一致。
2. `$_GET["3"] = "10"` **是取巧写法。** 原始 URL 参数是 `...&minify&3.0.10`，PHP 解析为 `$_GET["minify"]=""`（空字符串）、`$_GET["3"]="0.10"`。模拟不完全精确。
3. **没有处理** `$wave`**（波浪效果）参数。** 当前 `no_wave` 是写死的，如果后期主题设置启用了波浪效果，静态 CSS 需要重新生成。
4. **直接改主题** `functions.php`**，不可维护。** 主题更新会覆盖此修改。正确做法是：子主题（Child Theme）或自定义插件（Custom Plugin）来覆写 CSS URL。

***

## 阶段三：CDN 加速尝试

### Sakurairo 内置 CDN 选项

WordPress 后台 → 主题设置 → Performance → Resource Control：

- 关闭 `Provide Critical Frontend Resource locally` → 资源走 CDN
- `Public CDN Basepath` → `s.nmxc.ltd` 或 `jsDelivr`

### 踩坑记录

`s.nmxc.ltd`**：从本 VM 访问全部返回 404。** 原因推测是该 CDN 限制了境外 IP。

`jsDelivr`**：返回 HTML 目录列表，不是 CSS。** jsDelivr 是静态 CDN，不会执行 PHP 脚本。`css/?sakura_header&github...` 这个 URL 需要 PHP 处理，jsDelivr 只能返回 `css/` 目录下的文件列表。

**结论：** Sakurairo 的 CDN 选项**不能与 PHP 动态 CSS 配合使用**。正确的 CDN 使用方式：

1. 先生成静态 CSS 文件
2. 上传到 CDN 对象存储（如阿里云 OSS、又拍云）
3. 修改 `functions.php` 指向 CDN URL

> 备注：`s.nmxc.ltd` 是否对国内用户正常服务未验证。

***

## 阶段四：页面缓存

### WP Super Cache

状态：已安装启用，生成缓存文件。

**生效条件：**

不是"安装后在后台启用即可"。需要满足这些条件才能正常工作：

| 条件                                             | 检查方式                     |
| ---------------------------------------------- | ------------------------ |
| `wp-config.php` 中存在 `define('WP_CACHE', true)` | ✅ 已存在                    |
| `wp-content/advanced-cache.php` 存在             | ✅ WP Super Cache 安装时自动创建 |
| `wp-content/cache/` 目录可写                       | ✅ nginx 用户可写             |
| 插件设置页显示"缓存已启用"                                 | ✅                        |
| `wp-content/cache/supercache/` 下生成 `.html` 文件  | ✅ 各页面均已生成                |

**当前工作模式：** PHP 模式（`$wp_cache_mod_rewrite = 0`）。

- 每次请求仍会经过 PHP-FPM，但 WP Super Cache 拦截后直接读取缓存的 HTML 输出，不再执行完整的 WordPress 模板渲染和数据库查询。
- 实测 TTFB 约 **1.3-1.9s**（从本 VM），其中大部分是网络延迟而非 PHP 执行时间。

**PHP 模式 vs Mod_Rewrite 模式：**

| 模式             | TTFB（从本 VM） | 是否需要 Nginx 规则                 |
| -------------- | ----------- | ----------------------------- |
| PHP 模式（当前）     | ~1.5s       | 否                             |
| Mod_Rewrite 模式 | 预期 <0.1s    | 是，需在 nginx.conf 中加 rewrite 规则 |

Mod_Rewrite 模式尝试过但未成功配置，详见阶段五。

### Nginx FastCGI Cache

**尝试经过：**

1. 在 `http` 块添加 `fastcgi_cache_path` 和 `fastcgi_cache_key`
2. 在 `location ~ \.php$` 块添加 `fastcgi_cache WORDPRESS` 及相关指令
3. 添加 `fastcgi_ignore_headers` 以处理 WP Super Cache 输出的 `Cache-Control` 头
4. 用 `add_header X-FastCGI-Cache $upstream_cache_status` 调试

**现象：**

- Nginx 配置语法正确（`nginx -t` 通过）
- `systemctl reload nginx` 返回成功
- cache manager 进程出现（说明 `fastcgi_cache_path` 被加载）
- 但 `/tmp/nginx-cache/` 目录始终为空，响应头中 `X-FastCGI-Cache` 从未出现

**未生效的原因（未能完全确认）：**

根据 nginx 文档和实验现象，可能的根本原因有多个：

1. `try_files` **内部跳转与** `fastcgi_cache` **的作用域。** `location /` 中的 `try_files $uri $uri/ /index.php?$args` 会对 URI `/` 做内部跳转到 `/index.php`。nginx 会重新匹配 `location ~ \.php$`，理论上该 location 中的 `fastcgi_cache` 应生效。但实践中 `$upstream_cache_status` 一直为空，说明请求可能没有正常经过缓存逻辑处理。
2. **WP Super Cache 与 FastCGI Cache 的冲突。** WP Super Cache 在 PHP 层面通过 `advanced-cache.php` 处理请求，它可能会在 PHP-FPM 返回响应中写入 `Cache-Control: max-age=3, must-revalidate` 和 `Vary: Accept-Encoding, Cookie` 头。即使配置了 `fastcgi_ignore_headers`，多个复杂头组合仍可能触发 nginx 的缓存回避逻辑。
3. `Set-Cookie` **相关的缓存回避。** nginx 默认行为：如果后端返回 `Set-Cookie` 头，即使配置了 `fastcgi_ignore_headers Set-Cookie`，在某些版本或组合下仍可能绕过缓存。（本站响应中未发现 `Set-Cookie`，此条可能性较低。）

> 最终结论：由于排查时间有限，**未找到确切根因**，当前恢复到无 FastCGI Cache 的标准配置。后续如需深入排查，建议在测试环境逐个条件排除，并查看 nginx 的 error log（当前 rocky 用户无权限访问）。

***

## 阶段五：Nginx 配置

### 原始配置存在的问题

| 问题                               | 说明                                                                                                                    |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| WP Super Cache rewrite 规则位置错误    | 放在 `location /` 中，路径拼接产生双斜杠（`threcial.cn//index-https.html`），导致 404                                                   |
| 后改为 FastCGI Cache 配置             | 未生效，原因见阶段四                                                                                                            |
| PHP location 缺少 `try_files` 安全检查 | `location ~ \.php$` 中未加 `try_files $fastcgi_script_name =404`，存在远程执行风险（尽管 PHP-FPM 本身有 `security.limit_extensions` 防护） |

### 当前配置（简洁稳定版）

```nginx
location / {
    try_files $uri $uri/ /index.php?$args;
}

location ~ \.php$ {
    try_files $fastcgi_script_name =404;               # 安全：防止路径穿越

    include fastcgi_params;
    fastcgi_pass unix:/run/php-fpm/www.sock;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
}

location ^~ /xmlrpc.php {
    return 444;                                          # 安全：阻止 xmlrpc 攻击
}
```

> 注：当前运行配置**不含** `try_files $fastcgi_script_name =404;` —— 这是遗漏项，应补上。

***

## 阶段六：未闭环的优化项

以下问题被识别但**未实际解决**，不应标记为"已完成"：

| 项目                  | 当前状态   | 原因                                            |
| ------------------- | ------ | --------------------------------------------- |
| 📷 **图片压缩**         | ❌ 未执行  | 需要安装 ShortPixel/Smush 插件并批量压缩，用户未操作           |
| 🗂️ **静态资源缓存头**     | ❌ 未配置  | Nginx 缺少 `location ~* .(css                   |
| ⚡ **JS 加载优化**       | ❌ 未处理  | `app.js(69KB)` 和 `nav.js(55KB)` 仍从源站直出，应上 CDN |
| ☁️ **CSS/JS 上 CDN** | ❌ 未执行  | 静态 CSS 文件已生成，但未上传到 CDN 对象存储                   |
| 🔄 **更新静态 CSS**     | ❌ 无自动化 | 改主题颜色后需手动重新生成                                 |

***

## 安全与操作反思

### 备份失败未告知

修改 `/etc/nginx/nginx.conf` 时，`cp` 备份因 `/etc/nginx/` 目录无写权限失败，当时没有及时告知用户，导致：

- 配置无备份可回滚
- 用户对当前配置不知情

**教训：** 修改任何系统文件前，先确认备份成功。不成功则终止操作并通知用户。

### 直接修改主题文件

`functions.php` 属于主题文件，修改后会在主题更新时被覆盖。应使用子主题或自定义插件。

***

## 优化效果汇总（带测试口径说明）

以下数据均从本 VM（海外 Rocky Linux）测试。国内用户的真实体验应优于以下数值。

| 指标             | 优化前            | 优化后                         | 测试条件         |
| -------------- | -------------- | --------------------------- | ------------ |
| 首页 TTFB        | ~1.9s          | ~1.5-1.9s                   | 从本 VM 测，跨国链路 |
| CSS 服务器本地耗时    | 0.085s（PHP 动态） | 0.019s（静态文件）                | 服务器本地 curl   |
| 页面 CSS         | PHP 动态生成       | 静态文件 `sakurairo-static.css` | HTML 验证 ✅    |
| Gzip 压缩        | 已配置            | 已确认生效                       | 响应头确认 ✅      |
| WP Super Cache | 未安装            | 已安装，缓存文件已生成                 | cache目录确认 ✅  |
| 图片压缩           | 每张 500-800KB   | ❌ 未执行                       | —            |
| 静态资源缓存头        | ❌ 未配置          | ❌ 未配置                       | —            |
| JS 上 CDN       | ❌ 未处理          | ❌ 未处理                       | —            |

***

## 待办清单（按优先级排序）

```text
P0 图片压缩          → WordPress 后台装 ShortPixel/Smush 插件，批量跑一次
P1 静态资源缓存头     → nginx.conf 补上 expires 30d 配置
P1 CSS/JS 上 CDN     → 把 sakurairo-static.css 上传到 s.nmxc.ltd 或阿里云OSS
P2 JS 加载优化        → app.js/nav.js 异步加载或合并
P2 自动化刷新静态CSS  → 改主题颜色后一键 re-generate
```

***

## 关键词

`WordPress` `Sakurairo` `Nginx` `PHP动态CSS` `CSS静态化` `WP Super Cache` `FastCGI Cache` `CDN` `jsDelivr` `图片压缩` `子主题`
