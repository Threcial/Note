---
type: knowledge
status: evergreen
created: 2026-06-13
updated: 2026-06-13
related_to:
  - "[[Linux 运维基础]]"
  - "[[软件包管理]]"
has:
  - "[[rpm]]"
  - "[[yum]]"
  - "[[epel]]"
  - "[[源码安装]]"
  - "[[make]]"
_organized: true
---

# 软件包管理

## RPM

RPM 是 CentOS/RHEL 系软件包底层管理机制，安装、卸载、查询方便，但不能自动解决依赖。

### RPM 包命名

```text
dhcp-server-4.3.6-30.el8.x86_64.rpm
```

软件名-版本-编译次数.系统平台.架构.rpm

### RPM 命令

```bash
rpm -ivh dhcp-server-4.3.6-30.el8.x86_64.rpm   # 安装
rpm -e dhcp-server                                # 卸载
rpm -e --nodeps dhcp-server                       # 忽略依赖卸载
```

| 选项 | 含义 |
| --- | --- |
| `-q` | 查询是否安装 |
| `-qi` | 查看软件包信息 |
| `-ql` | 查看文件列表 |
| `-qa` | 查看所有已安装包 |
| `-qf /path/file` | 查询文件属于哪个包 |
| `--nodeps` | 忽略依赖（慎用） |
| `--test` | 测试安装 |

## YUM

YUM 基于 RPM，核心价值是自动处理依赖和仓库管理。

### 配置网络源

```bash
curl -s -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum clean all && yum makecache
```

### EPEL

```bash
yum install epel-release -y
curl -s -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
```

### 常用命令

```bash
yum install 包 -y
yum remove 包 -y
yum list | grep xxx
yum info 包
yum search 关键词
yum update 包 -y
yum repolist
yum groupinstall "Development tools" -y
```

### 常用目录

| 路径 | 作用 |
| --- | --- |
| `/etc/yum.conf` | 主配置文件 |
| `/etc/yum.repos.d` | 仓库配置文件目录 |
| `/var/cache/yum` | 缓存目录 |
| `/var/log/yum.log` | 日志 |

## 源码安装

适合 yum 版本过低或需要自定义编译参数的场景。

```bash
cd 源码目录
./configure --prefix=安装目录
make
make install
make clean
```

| 步骤 | 作用 |
| --- | --- |
| `./configure` | 检测环境，生成 Makefile |
| `make` | 编译 |
| `make install` | 安装 |
| `make clean` | 清理临时文件 |

卸载源码安装的软件通常直接删除安装目录。

## 易错点

| 场景 | 说明 |
| --- | --- |
| rpm 忽略依赖安装 | 包可能装上但无法运行 |
| yum 源更新后不 makecache | 安装可能慢或读旧缓存 |
| `yum update -y` 随便全量升级 | 生产环境可能引入版本变化风险 |
| 源码安装后不知道文件在哪 | 编译前明确 `--prefix` |
