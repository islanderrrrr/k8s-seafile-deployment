# Kubernetes Seafile 部署配置

在 Kubernetes 集群上部署 Seafile 私有云存储系统的完整配置文件。

## 项目简介

本项目提供了在 Kubernetes 集群上部署 Seafile 的完整配置文件，包括：
- **MariaDB**: 数据库服务
- **Memcached**: 缓存服务
- **Seafile**: 文件同步和共享服务

## 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes 集群架构                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   k8s-master    │  │   k8s-node1     │  │   k8s-node2     │ │
│  │ 172.22.175.252  │  │ 172.22.174.244  │  │ 172.22.165.101  │ │
│  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤ │
│  │   控制平面       │  │    MariaDB      │  │    Seafile      │ │
│  │  - API Server   │  │   Memcached     │  │                 │ │
│  │  - etcd         │  │    CoreDNS      │  │    CoreDNS      │ │
│  │  - Controller   │  │                 │  │                 │ │
│  │  - Scheduler    │  │  /data/mariadb/ │  │  /data/seafile/ │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                   │
│  Pod 网络 (Flannel): 10.244.0.0/16                              │
│  Service 网络: 10.96.0.0/12                                      │
│  NodePort: 30080                                                 │
└─────────────────────────────────────────────────────────────────┘

       ┌──────────────────────────────────────┐
       │  外部访问: http://172.22.175.252:30080 │
       └──────────────────────────────────────┘
```

## 前置要求

### 硬件要求
- 最少 3 个节点的 Kubernetes 集群（1 个 Master + 2 个 Worker）
- 每个节点至少 2 CPU 核心，4GB 内存
- 足够的存储空间：
  - k8s-node1: 至少 5GB（MariaDB 数据）
  - k8s-node2: 至少 10GB（Seafile 数据）

### 软件要求
- Kubernetes 1.20+
- kubectl 命令行工具
- Flannel CNI 网络插件

### 存储准备
在相应节点上创建本地存储目录：

```bash
# 在 k8s-node1 上执行
sudo mkdir -p /data/mariadb
sudo chmod 777 /data/mariadb

# 在 k8s-node2 上执行
sudo mkdir -p /data/seafile
sudo chmod 777 /data/seafile
```

## 快速部署步骤

### 1. 创建命名空间
```bash
kubectl apply -f manifests/namespace.yaml
```

### 2. 创建 Secrets
```bash
kubectl apply -f manifests/secrets.yaml
```

### 3. 部署 MariaDB
```bash
kubectl apply -f manifests/mariadb/pv.yaml
kubectl apply -f manifests/mariadb/pvc.yaml
kubectl apply -f manifests/mariadb/deployment.yaml
kubectl apply -f manifests/mariadb/service.yaml
```

等待 MariaDB Pod 就绪：
```bash
kubectl wait --for=condition=ready pod -l app=mariadb -n seafile --timeout=300s
```

### 4. 部署 Memcached
```bash
kubectl apply -f manifests/memcached/deployment.yaml
kubectl apply -f manifests/memcached/service.yaml
```

### 5. 部署 Seafile
```bash
kubectl apply -f manifests/seafile/pv.yaml
kubectl apply -f manifests/seafile/pvc.yaml
kubectl apply -f manifests/seafile/deployment.yaml
kubectl apply -f manifests/seafile/service.yaml
```

等待 Seafile Pod 就绪：
```bash
kubectl wait --for=condition=ready pod -l app=seafile -n seafile --timeout=600s
```

### 一键部署（可选）
```bash
# 按顺序应用所有配置
kubectl apply -f manifests/namespace.yaml
kubectl apply -f manifests/secrets.yaml
kubectl apply -f manifests/mariadb/
kubectl apply -f manifests/memcached/
kubectl apply -f manifests/seafile/
```

## 资源分布图

| 节点 | IP | 角色 | 运行的 Pod | 本地存储 |
|------|-----|------|-----------|---------|
| k8s-master | 172.22.175.252 | 控制平面 | API Server, etcd, Controller, Scheduler | 配置文件 |
| k8s-node1 | 172.22.174.244 | 工作节点 | MariaDB, Memcached, CoreDNS | /data/mariadb/ |
| k8s-node2 | 172.22.165.101 | 工作节点 | Seafile, CoreDNS | /data/seafile/ |

## 网络配置

### 网络参数
- **Pod 网络 (Flannel CNI)**: 10.244.0.0/16
- **Service 网络**: 10.96.0.0/12
- **NodePort**: 30080（Seafile Web UI）

### 服务端点
- **MariaDB Service**: mariadb.seafile.svc.cluster.local:3306
- **Memcached Service**: memcached.seafile.svc.cluster.local:11211
- **Seafile Service**: seafile.seafile.svc.cluster.local:80

## 访问信息

### Web 访问
- **URL**: http://172.22.175.252:30080
- 通过任意节点 IP + NodePort 30080 访问

### 默认登录凭据
- **管理员邮箱**: admin@seafile.local
- **管理员密码**: admin_password

⚠️ **安全警告**: 生产环境中请务必修改默认密码！

## 常用命令

### 查看所有资源
```bash
kubectl get all -n seafile
```

### 查看 Pod 状态
```bash
kubectl get pods -n seafile -o wide
```

### 查看 PV/PVC
```bash
kubectl get pv
kubectl get pvc -n seafile
```

### 查看服务
```bash
kubectl get svc -n seafile
```

### 查看 Pod 日志
```bash
# MariaDB 日志
kubectl logs -f deployment/mariadb -n seafile

# Memcached 日志
kubectl logs -f deployment/memcached -n seafile

# Seafile 日志
kubectl logs -f deployment/seafile -n seafile
```

### 进入 Pod 调试
```bash
# 进入 Seafile Pod
kubectl exec -it deployment/seafile -n seafile -- /bin/bash

# 进入 MariaDB Pod
kubectl exec -it deployment/mariadb -n seafile -- /bin/bash
```

### 重启服务
```bash
# 重启 Seafile
kubectl rollout restart deployment/seafile -n seafile

# 重启 MariaDB
kubectl rollout restart deployment/mariadb -n seafile

# 重启 Memcached
kubectl rollout restart deployment/memcached -n seafile
```

### 扩缩容（注意：有状态服务扩缩容需谨慎）
```bash
# 查看副本数
kubectl get deployment -n seafile

# 扩展 Memcached（无状态，可以扩展）
kubectl scale deployment/memcached --replicas=2 -n seafile
```

### 删除部署
```bash
# 删除所有资源
kubectl delete namespace seafile

# 或者逐个删除
kubectl delete -f manifests/seafile/
kubectl delete -f manifests/memcached/
kubectl delete -f manifests/mariadb/
kubectl delete -f manifests/secrets.yaml
kubectl delete -f manifests/namespace.yaml

# 删除 PV（注意：这会导致数据丢失）
kubectl delete pv mariadb-pv seafile-pv
```

## 注意事项

### 安全建议
1. **修改默认密码**: 部署后立即修改 `manifests/secrets.yaml` 中的密码
   ```bash
   kubectl edit secret seafile-secrets -n seafile
   ```

2. **使用强密码**: 确保数据库和管理员密码足够复杂

3. **网络隔离**: 生产环境建议使用 NetworkPolicy 限制 Pod 间通信

4. **启用 HTTPS**: 使用 Ingress + TLS 证书保护 Web 访问

### 存储注意事项
1. **数据持久化**: 使用的是 local storage，数据绑定在特定节点上
2. **备份策略**: 定期备份 `/data/mariadb` 和 `/data/seafile` 目录
3. **存储空间**: 监控存储使用情况，及时扩容
4. **权限问题**: 确保存储目录有正确的访问权限（777 或特定用户）

### 运维建议
1. **监控**: 部署 Prometheus + Grafana 监控集群和应用状态
2. **日志**: 配置日志收集系统（如 ELK、Loki）
3. **告警**: 设置资源使用、Pod 状态等告警规则
4. **升级**: 升级前务必备份数据，测试兼容性

### 性能优化
1. **Memcached**: 根据实际需求调整内存大小（当前 256MB）
2. **MariaDB**: 优化数据库配置，增加连接池大小
3. **Seafile**: 根据用户数量调整并发连接数

## 文档

- [架构设计说明](docs/architecture.md)
- [故障排查指南](docs/troubleshooting.md)

## 版本信息

- Kubernetes: 1.20+
- MariaDB: 10.11
- Memcached: 1.6.18
- Seafile: 11.0-latest

## 许可证

本项目仅供学习和参考使用。

## 贡献

欢迎提交 Issue 和 Pull Request！
