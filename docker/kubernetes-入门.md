---
type: knowledge
status: evergreen
created: 2026-06-14
updated: 2026-06-14
related_to:
  - "[[docker-compose-入门]]"
  - "[[dockerfile-参考]]"
  - "[[容器编排]]"
  - "[[Kubernetes]]"
_organized: true
---

# Kubernetes 入门

Compose 适合单机多容器项目，Kubernetes 适合集群级容器编排。K8s 关注的不只是启动容器，还包括调度、服务发现、滚动更新、自动恢复、水平扩容、配置和密钥管理。

## 与 Compose 对比

| Compose | Kubernetes |
| --- | --- |
| 单机项目编排 | 多节点集群编排 |
| `services` | `Deployment` / `StatefulSet` / `Service` |
| `volumes` | `PersistentVolume` / `PersistentVolumeClaim` |
| `networks` | CNI 网络和 Service 发现 |
| `restart` | 控制器自动拉起 Pod |

## 学习路径

先把 [[dockerfile-参考]] 和 [[docker-compose-入门]] 写熟，再按以下顺序学习 K8s：

1. Pod — 最小调度单元
2. Deployment — 声明式副本管理
3. Service — 网络暴露与服务发现
4. ConfigMap & Secret — 配置与密钥管理
5. Ingress — 七层入口
6. PersistentVolume & PersistentVolumeClaim — 持久化存储
