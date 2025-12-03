# Seafile Kubernetes 架构设计

## 1. 整体架构设计

本部署方案采用三层架构设计，将 Seafile 私有云存储系统部署在 Kubernetes 集群上：

```
┌────────────────────────────────────────────────────────────┐
│                         用户层                              │
│                                                            │
│    浏览器/客户端 ──> http://172.22.175.252:30080           │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│                      服务层 (Service)                       │
│                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │   Seafile    │  │   MariaDB    │  │  Memcached   │    │
│  │   Service    │  │   Service    │  │   Service    │    │
│  │  NodePort    │  │  ClusterIP   │  │  ClusterIP   │    │
│  │   :30080     │  │   :3306      │  │   :11211     │    │
│  └──────────────┘  └──────────────┘  └──────────────┘    │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│                      应用层 (Pod)                           │
│                                                            │
│  ┌────────────────┐ ┌─────────────┐ ┌──────────────┐     │
│  │  Seafile Pod   │ │ MariaDB Pod │ │ Memcached Pod│     │
│  │   k8s-node2    │ │  k8s-node1  │ │  k8s-node1   │     │
│  │                │ │             │ │              │     │
│  │  seafileltd/   │ │ mariadb:    │ │ memcached:   │     │
│  │  seafile-mc    │ │   10.11     │ │   1.6.18     │     │
│  │  :11.0-latest  │ │             │ │              │     │
│  └────────────────┘ └─────────────┘ └──────────────┘     │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│                     存储层 (Storage)                        │
│                                                            │
│  ┌────────────────┐           ┌────────────────┐          │
│  │  Seafile PV    │           │  MariaDB PV    │          │
│  │  /data/seafile │           │ /data/mariadb  │          │
│  │   k8s-node2    │           │   k8s-node1    │          │
│  │     10Gi       │           │      5Gi       │          │
│  └────────────────┘           └────────────────┘          │
└────────────────────────────────────────────────────────────┘
```

### 架构特点

1. **分层解耦**: 服务层、应用层、存储层相互独立，便于维护和扩展
2. **节点亲和性**: 使用 `nodeSelector` 将有状态服务固定到特定节点
3. **持久化存储**: 使用 PV/PVC 确保数据持久化
4. **服务发现**: 通过 Kubernetes Service 实现自动服务发现和负载均衡

## 2. 各组件功能介绍

### 2.1 MariaDB (数据库服务)

**功能**: 存储 Seafile 的元数据和用户信息

**关键特性**:
- 版本: MariaDB 10.11
- 端口: 3306
- 存储: 5Gi 本地存储（PV）
- 部署位置: k8s-node1
- Service 类型: ClusterIP (headless service, clusterIP: None)

**环境变量**:
- `MYSQL_ROOT_PASSWORD`: 数据库 root 密码（从 Secret 获取）
- `MYSQL_LOG_CONSOLE`: 启用控制台日志输出

**数据持久化**:
- 挂载点: `/var/lib/mysql`
- 本地路径: `/data/mariadb` (k8s-node1)
- 回收策略: Retain (保留数据)

### 2.2 Memcached (缓存服务)

**功能**: 提供内存缓存，提升 Seafile 性能

**关键特性**:
- 版本: Memcached 1.6.18
- 端口: 11211
- 内存大小: 256MB
- 部署位置: k8s-node1（无节点选择器，调度器自动分配）
- Service 类型: ClusterIP

**启动参数**:
- `-m 256`: 分配 256MB 内存

**特点**:
- 无状态服务，无需持久化存储
- 可以横向扩展（如需要）
- 作为 Seafile 的缓存层，减少数据库访问压力

### 2.3 Seafile (文件同步和共享服务)

**功能**: 提供文件同步、共享和协作功能

**关键特性**:
- 版本: Seafile MC 11.0-latest
- 端口: 80
- 存储: 10Gi 本地存储（PV）
- 部署位置: k8s-node2
- Service 类型: NodePort (端口 30080)

**环境变量**:
- `DB_HOST`: MariaDB 服务地址 (mariadb)
- `DB_ROOT_PASSWD`: 数据库密码
- `SEAFILE_ADMIN_EMAIL`: 管理员邮箱
- `SEAFILE_ADMIN_PASSWORD`: 管理员密码
- `SEAFILE_SERVER_HOSTNAME`: 服务器主机名/IP
- `TIME_ZONE`: 时区设置 (Asia/Shanghai)

**数据持久化**:
- 挂载点: `/shared`
- 本地路径: `/data/seafile` (k8s-node2)
- 存储内容: 用户上传的文件、配置文件等

## 3. 数据流向图

### 3.1 用户访问流程

```
┌─────────┐
│  用户    │
└────┬────┘
     │ HTTP Request
     │ http://172.22.175.252:30080
     ▼
┌─────────────────┐
│   k8s-master    │
│  (NodePort)     │
│    :30080       │
└────┬────────────┘
     │ 转发到
     ▼
┌─────────────────┐
│ Seafile Service │
│  (NodePort)     │
│   Port: 80      │
└────┬────────────┘
     │ 负载均衡
     ▼
┌─────────────────┐
│  Seafile Pod    │
│   k8s-node2     │
│   Port: 80      │
└─────────────────┘
```

### 3.2 Seafile 内部数据流

```
┌─────────────────┐
│  Seafile Pod    │
└────┬────┬───────┘
     │    │
     │    └──────────────────┐
     │                       │
     │ 数据库操作             │ 缓存操作
     ▼                       ▼
┌─────────────────┐    ┌──────────────────┐
│  MariaDB Pod    │    │  Memcached Pod   │
│   Port: 3306    │    │   Port: 11211    │
└────┬────────────┘    └──────────────────┘
     │
     │ 持久化
     ▼
┌─────────────────┐
│  MariaDB PV     │
│ /data/mariadb   │
│   k8s-node1     │
└─────────────────┘
```

### 3.3 文件上传流程

```
用户浏览器
    │
    │ 1. 上传文件
    ▼
Seafile Pod (k8s-node2)
    │
    ├─→ 2. 写入元数据 → MariaDB Pod → MariaDB PV
    │                    (k8s-node1)    (/data/mariadb)
    │
    ├─→ 3. 更新缓存 → Memcached Pod
    │                  (k8s-node1)
    │
    └─→ 4. 保存文件 → Seafile PV
                      (/data/seafile on k8s-node2)
```

## 4. 存储架构说明

### 4.1 存储类型

本方案使用 **Local Storage (本地存储)** 作为持久化存储方案：

**优点**:
- 性能优异，直接访问本地磁盘
- 配置简单，无需外部存储系统
- 适合小型部署和开发测试

**缺点**:
- 数据绑定在特定节点，不支持跨节点迁移
- 节点故障会导致数据不可用
- 不支持动态扩容

### 4.2 PV/PVC 架构

```
┌────────────────────────────────────────────────────────┐
│                  PersistentVolume (PV)                  │
│                      集群级资源                         │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌────────────────────┐    ┌────────────────────┐    │
│  │    mariadb-pv      │    │    seafile-pv      │    │
│  ├────────────────────┤    ├────────────────────┤    │
│  │ 容量: 5Gi          │    │ 容量: 10Gi         │    │
│  │ 路径: /data/mariadb│    │ 路径: /data/seafile│    │
│  │ 节点: k8s-node1    │    │ 节点: k8s-node2    │    │
│  │ 策略: Retain       │    │ 策略: Retain       │    │
│  └────────────────────┘    └────────────────────┘    │
└────────────────────────────────────────────────────────┘
                    │                    │
                    │ Bound              │ Bound
                    ▼                    ▼
┌────────────────────────────────────────────────────────┐
│            PersistentVolumeClaim (PVC)                  │
│                   命名空间级资源                        │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌────────────────────┐    ┌────────────────────┐    │
│  │   mariadb-pvc      │    │   seafile-pvc      │    │
│  ├────────────────────┤    ├────────────────────┤    │
│  │ 请求: 5Gi          │    │ 请求: 10Gi         │    │
│  │ 命名空间: seafile  │    │ 命名空间: seafile  │    │
│  └────────────────────┘    └────────────────────┘    │
└────────────────────────────────────────────────────────┘
                    │                    │
                    │ Mount              │ Mount
                    ▼                    ▼
┌────────────────────────────────────────────────────────┐
│                        Pod                              │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌────────────────────┐    ┌────────────────────┐    │
│  │   MariaDB Pod      │    │   Seafile Pod      │    │
│  ├────────────────────┤    ├────────────────────┤    │
│  │ /var/lib/mysql     │    │ /shared            │    │
│  └────────────────────┘    └────────────────────┘    │
└────────────────────────────────────────────────────────┘
```

### 4.3 存储配置详情

| 资源 | 容量 | 访问模式 | 回收策略 | 绑定节点 | 本地路径 |
|------|------|---------|---------|---------|---------|
| mariadb-pv | 5Gi | ReadWriteOnce | Retain | k8s-node1 | /data/mariadb |
| seafile-pv | 10Gi | ReadWriteOnce | Retain | k8s-node2 | /data/seafile |

**回收策略说明**:
- `Retain`: PVC 删除后保留 PV 和数据，需要手动清理

**访问模式说明**:
- `ReadWriteOnce`: 只能被单个节点以读写方式挂载

### 4.4 节点亲和性

使用 `nodeAffinity` 确保 PV 绑定到正确的节点：

```yaml
nodeAffinity:
  required:
    nodeSelectorTerms:
    - matchExpressions:
      - key: kubernetes.io/hostname
        operator: In
        values:
        - k8s-node1  # 或 k8s-node2
```

同时在 Deployment 中使用 `nodeSelector` 确保 Pod 调度到对应节点：

```yaml
nodeSelector:
  kubernetes.io/hostname: k8s-node1  # 或 k8s-node2
```

## 5. 网络通信说明

### 5.1 网络拓扑

```
┌─────────────────────────────────────────────────────────┐
│                   Flannel CNI                            │
│                Pod 网络: 10.244.0.0/16                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │ k8s-node1   │  │ k8s-node2   │  │ k8s-master  │    │
│  │ 10.244.1.0/ │  │ 10.244.2.0/ │  │ 10.244.0.0/ │    │
│  │    24       │  │    24       │  │    24       │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                Service 网络                              │
│                   10.96.0.0/12                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  mariadb.seafile.svc.cluster.local    → 10.96.x.x:3306 │
│  memcached.seafile.svc.cluster.local  → 10.96.x.x:11211│
│  seafile.seafile.svc.cluster.local    → 10.96.x.x:80   │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                 Node 网络 (物理网络)                     │
│                   172.22.0.0/16                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  k8s-master → 172.22.175.252                           │
│  k8s-node1  → 172.22.174.244                           │
│  k8s-node2  → 172.22.165.101                           │
└─────────────────────────────────────────────────────────┘
```

### 5.2 Service 类型说明

#### ClusterIP (内部服务)
- **MariaDB Service**: `mariadb.seafile.svc.cluster.local:3306`
  - 仅集群内部访问
  - Headless Service (clusterIP: None)
  - 直接解析到 Pod IP
  
- **Memcached Service**: `memcached.seafile.svc.cluster.local:11211`
  - 仅集群内部访问
  - 标准 ClusterIP
  - 通过虚拟 IP 访问

#### NodePort (外部访问)
- **Seafile Service**: 
  - 集群内: `seafile.seafile.svc.cluster.local:80`
  - 集群外: `<任意节点IP>:30080`
  - NodePort: 30080
  - 映射到 Pod 的 80 端口

### 5.3 通信流程

#### Seafile → MariaDB
```
Seafile Pod (10.244.2.x:随机)
    │
    │ DNS 解析: mariadb → Pod IP
    ▼
MariaDB Service (mariadb.seafile.svc.cluster.local)
    │
    │ 直接路由（Headless Service）
    ▼
MariaDB Pod (10.244.1.x:3306)
```

#### Seafile → Memcached
```
Seafile Pod (10.244.2.x:随机)
    │
    │ DNS 解析: memcached → ClusterIP
    ▼
Memcached Service (10.96.x.x:11211)
    │
    │ iptables/IPVS 转发
    ▼
Memcached Pod (10.244.1.x:11211)
```

#### 外部用户 → Seafile
```
用户浏览器
    │
    │ HTTP: http://172.22.175.252:30080
    ▼
任意 K8s 节点 (172.22.x.x:30080)
    │
    │ kube-proxy 转发
    ▼
Seafile Service (10.96.x.x:80)
    │
    │ iptables/IPVS 转发
    ▼
Seafile Pod (10.244.2.x:80)
```

### 5.4 DNS 解析

CoreDNS 提供集群内 DNS 解析：

| 服务名 | FQDN | 解析结果 |
|--------|------|---------|
| mariadb | mariadb.seafile.svc.cluster.local | Pod IP (Headless) |
| memcached | memcached.seafile.svc.cluster.local | ClusterIP |
| seafile | seafile.seafile.svc.cluster.local | ClusterIP |

**简写形式** (在 seafile 命名空间内):
- `mariadb` → `mariadb.seafile.svc.cluster.local`
- `memcached` → `memcached.seafile.svc.cluster.local`
- `seafile` → `seafile.seafile.svc.cluster.local`

### 5.5 防火墙和安全组

需要开放的端口：
- **30080**: NodePort，供外部访问 Seafile Web UI
- **6443**: API Server（Master 节点）
- **10250**: Kubelet API（所有节点）
- **2379-2380**: etcd（Master 节点）

## 6. 高可用性考虑

### 6.1 当前架构的局限性

- **单点故障**: 所有组件都是单副本，任何一个组件故障都会影响服务
- **本地存储**: 数据绑定到特定节点，节点故障导致数据不可用
- **无自动故障恢复**: 节点故障后需要手动干预

### 6.2 高可用改进建议

1. **数据库高可用**:
   - 使用 MariaDB Galera Cluster（多主复制）
   - 或使用云提供商的托管数据库服务

2. **存储高可用**:
   - 使用分布式存储（Ceph、GlusterFS）
   - 或使用云存储（EBS、云盘）
   - 实现存储级别的数据复制

3. **应用高可用**:
   - Seafile 扩展到多副本（需要共享存储支持）
   - Memcached 扩展到多副本提高缓存命中率

4. **负载均衡**:
   - 使用 Ingress Controller 替代 NodePort
   - 配置多个 Ingress 副本实现高可用

## 7. 扩展性考虑

### 7.1 垂直扩展（增加资源）

修改 Deployment 的资源请求和限制：

```yaml
resources:
  requests:
    memory: "2Gi"
    cpu: "1000m"
  limits:
    memory: "4Gi"
    cpu: "2000m"
```

### 7.2 水平扩展（增加副本）

- **Memcached**: 可以直接扩展副本数
- **Seafile**: 需要共享存储支持多副本部署
- **MariaDB**: 需要配置主从复制或集群模式

### 7.3 存储扩展

1. **扩容 PV**:
   ```bash
   # 删除 PVC（不会删除数据）
   kubectl delete pvc seafile-pvc -n seafile
   
   # 修改 PV 容量
   kubectl edit pv seafile-pv
   
   # 重新创建 PVC
   kubectl apply -f manifests/seafile/pvc.yaml
   ```

2. **迁移到更大的存储**:
   - 准备新的存储目录
   - 复制数据到新目录
   - 更新 PV 配置
   - 重启相关 Pod

## 8. 总结

本架构设计遵循 Kubernetes 最佳实践，实现了：
- ✅ 服务的容器化部署
- ✅ 数据的持久化存储
- ✅ 服务间的网络隔离和通信
- ✅ 通过 NodePort 对外提供服务

适用场景：
- 小型团队的私有云存储
- 开发和测试环境
- 学习 Kubernetes 部署实践

后续改进方向：
- 实现高可用架构
- 使用分布式存储
- 添加监控和告警
- 实现自动备份和恢复