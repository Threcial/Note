---
type: Practice
status: Organized
Belongs to: "[[Docker]]"
Related to:
  - "[[docker-基础概念]]"
source: ""
created: 2026-06-14
updated: 2026-06-14
---

# Docker 安装与配置

## CentOS 7 安装

```bash
yum remove -y docker*
yum install -y yum-utils
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io
systemctl start docker
systemctl enable docker
docker version
docker info
```

如果 compose 插件依赖安装失败，可以手动下载对应 rpm 安装。

## 镜像加速配置

```bash
mkdir -p /etc/docker
cat >/etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1panel.live/",
    "https://dockerhub.icu",
    "https://docker.awsl9527.cn",
    "https://dockerpull.com"
  ]
}
EOF
systemctl daemon-reload
systemctl restart docker
```

镜像站可用性会变化，拉取失败时需要更换源或使用自己的镜像加速地址。
