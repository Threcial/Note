---
type: Concept
status: Organized
Belongs to: "[[高可用架构]]"
Related to:
  - "[[nginx-优化与负载均衡]]"
Has:
  - "[[Keepalived]]"
  - "[[VRRP]]"
  - "[[VIP]]"
source: day18.md
created: 2026-06-14
updated: 2026-06-14
---

# Keepalived 高可用

Nginx 分发器本身可能成为单点。Keepalived 通过 VRRP 维护一个 VIP，让主节点故障后备节点接管。

云主机通常不支持自建 VIP 漂移，应优先使用云厂商 SLB/CLB/ELB。

## MASTER 配置

```nginx
global_defs {
    router_id wb01
}

vrrp_script chk_nginx {
    script "/etc/keepalived/nginx_check.sh"
    interval 1
    weight -50
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 33
    priority 150
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1111
    }

    track_script {
        chk_nginx
    }

    virtual_ipaddress {
        192.168.31.33/24 dev ens33 label ens33:3
    }
}
```

## BACKUP 配置

```nginx
global_defs {
    router_id wb02
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 33
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass 1111
    }

    virtual_ipaddress {
        192.168.31.33/24 dev ens33 label ens33:3
    }
}
```

## 关键参数

| 参数 | 要求 |
| --- | --- |
| `router_id` | 每台机器唯一 |
| `virtual_router_id` | 同一组必须一致 |
| `priority` | MASTER 高于 BACKUP |
| `auth_pass` | 同组一致 |
| `virtual_ipaddress` | 两端一致 |

## Nginx 检测脚本

```bash
#!/bin/bash
if ! pidof nginx >/dev/null 2>&1; then
    exit 1
fi
exit 0
```

```bash
chmod +x /etc/keepalived/nginx_check.sh
systemctl enable --now keepalived
```

## 裂脑

两台节点同时持有 VIP。常见原因：心跳网络异常、防火墙拦截 VRRP（协议号 112）、配置不一致。

监控思路：备节点上检测主节点服务是否可达，如果主节点 80 端口可达且备节点持有 VIP，触发报警。

## 易错点

| 问题 | 处理 |
| --- | --- |
| Keepalived 云主机不漂移 | 云环境通常不支持自建 VIP |
| 裂脑 | 检查 VRRP 通信、防火墙、心跳链路 |
