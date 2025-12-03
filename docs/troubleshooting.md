# Seafile Kubernetes 故障排查指南

本文档提供 Seafile 在 Kubernetes 上部署时常见问题的排查方法和解决方案。

## 目录

1. [Pod 启动失败排查](#1-pod-启动失败排查)
2. [数据库连接问题](#2-数据库连接问题)
3. [Seafile 无法访问问题](#3-seafile-无法访问问题)
4. [存储权限问题](#4-存储权限问题)
5. [日志查看方法](#5-日志查看方法)
6. [常用调试命令](#6-常用调试命令)

---

## 1. Pod 启动失败排查

### 1.1 查看 Pod 状态

```bash
# 查看所有 Pod 状态
kubectl get pods -n seafile

# 查看详细信息
kubectl get pods -n seafile -o wide

# 查看 Pod 事件
kubectl describe pod <pod-name> -n seafile
```

### 1.2 常见 Pod 状态及处理

#### ImagePullBackOff / ErrImagePull

**症状**: Pod 状态显示 `ImagePullBackOff` 或 `ErrImagePull`

**原因**:
- 镜像名称错误
- 镜像不存在
- 网络问题无法拉取镜像
- 私有镜像仓库认证失败

**解决方案**:

```bash
# 查看详细错误信息
kubectl describe pod <pod-name> -n seafile

# 检查镜像是否正确
kubectl get pod <pod-name> -n seafile -o jsonpath='{.spec.containers[*].image}'

# 在节点上手动拉取镜像测试
docker pull mariadb:10.11
docker pull memcached:1.6.18
docker pull seafileltd/seafile-mc:11.0-latest

# 如果是网络问题，配置镜像加速器
# 编辑 /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://registry.docker-cn.com"
  ]
}
```

#### CrashLoopBackOff

**症状**: Pod 反复重启，状态为 `CrashLoopBackOff`

**原因**:
- 应用启动失败
- 配置错误
- 依赖服务未就绪
- 资源不足

**解决方案**:

```bash
# 查看 Pod 日志
kubectl logs <pod-name> -n seafile
kubectl logs <pod-name> -n seafile --previous  # 查看上一次运行的日志

# 查看 Pod 重启次数
kubectl get pod <pod-name> -n seafile -o jsonpath='{.status.containerStatuses[0].restartCount}'

# 针对 MariaDB CrashLoopBackOff
# 1. 检查存储目录权限
ssh k8s-node1
ls -la /data/mariadb
sudo chmod 777 /data/mariadb

# 2. 检查是否有其他进程占用
sudo lsof /data/mariadb

# 3. 检查磁盘空间
df -h /data/mariadb

# 针对 Seafile CrashLoopBackOff
# 1. 确保 MariaDB 已经就绪
kubectl wait --for=condition=ready pod -l app=mariadb -n seafile --timeout=300s

# 2. 检查数据库密码是否匹配
kubectl get secret seafile-secrets -n seafile -o jsonpath='{.data.mysql-root-password}' | base64 -d

# 3. 检查存储挂载
kubectl describe pod <seafile-pod-name> -n seafile | grep -A 10 "Volumes:"
```

#### Pending

**症状**: Pod 一直处于 `Pending` 状态

**原因**:
- 没有可用的节点满足调度条件
- 资源不足（CPU、内存）
- PVC 未绑定
- NodeSelector 或亲和性配置错误

**解决方案**:

```bash
# 查看为什么 Pod 无法调度
kubectl describe pod <pod-name> -n seafile

# 检查节点资源
kubectl top nodes
kubectl describe node <node-name>

# 检查 PVC 绑定状态
kubectl get pvc -n seafile
kubectl describe pvc <pvc-name> -n seafile

# 检查 PV 状态
kubectl get pv
kubectl describe pv <pv-name>

# 检查节点标签
kubectl get nodes --show-labels

# 如果是 NodeSelector 问题，为节点添加标签
kubectl label nodes k8s-node1 kubernetes.io/hostname=k8s-node1
kubectl label nodes k8s-node2 kubernetes.io/hostname=k8s-node2
```

#### Init:Error / Init:CrashLoopBackOff

**症状**: Init 容器失败

**解决方案**:

```bash
# 查看 Init 容器日志
kubectl logs <pod-name> -n seafile -c <init-container-name>

# 查看 Init 容器状态
kubectl describe pod <pod-name> -n seafile
```

### 1.3 PVC 绑定问题

**症状**: PVC 一直处于 `Pending` 状态

```bash
# 检查 PVC 状态
kubectl get pvc -n seafile
kubectl describe pvc mariadb-pvc -n seafile
kubectl describe pvc seafile-pvc -n seafile

# 检查 PV 状态
kubectl get pv
kubectl describe pv mariadb-pv
kubectl describe pv seafile-pv

# 常见问题：
# 1. PV 不存在 - 先创建 PV
kubectl apply -f manifests/mariadb/pv.yaml
kubectl apply -f manifests/seafile/pv.yaml

# 2. StorageClass 不匹配
kubectl get storageclass

# 3. 容量不匹配
# PVC 请求的容量不能大于 PV 提供的容量

# 4. AccessMode 不匹配
# 确保 PV 和 PVC 的 accessModes 一致

# 5. 节点亲和性问题
# 确保 PV 的 nodeAffinity 中的节点存在且标签正确
```

---

## 2. 数据库连接问题

### 2.1 Seafile 无法连接 MariaDB

**症状**: Seafile 日志显示数据库连接错误

```bash
# 查看 Seafile 日志
kubectl logs -f deployment/seafile -n seafile | grep -i "database\|mysql\|mariadb"
```

**可能的错误信息**:
- `Can't connect to MySQL server`
- `Access denied for user`
- `Unknown database`

**排查步骤**:

#### 步骤 1: 检查 MariaDB 是否就绪

```bash
# 检查 MariaDB Pod 状态
kubectl get pods -n seafile -l app=mariadb

# 检查 MariaDB 日志
kubectl logs -f deployment/mariadb -n seafile

# 等待 MariaDB 就绪
kubectl wait --for=condition=ready pod -l app=mariadb -n seafile --timeout=300s
```

#### 步骤 2: 检查 Service 配置

```bash
# 检查 Service 是否存在
kubectl get svc -n seafile

# 检查 Service 详情
kubectl describe svc mariadb -n seafile

# 测试 DNS 解析
kubectl run -it --rm debug --image=busybox --restart=Never -n seafile -- nslookup mariadb
kubectl run -it --rm debug --image=busybox --restart=Never -n seafile -- nslookup mariadb.seafile.svc.cluster.local
```

#### 步骤 3: 检查网络连通性

```bash
# 从 Seafile Pod 测试连接 MariaDB
kubectl exec -it deployment/seafile -n seafile -- ping mariadb

# 测试端口连通性
kubectl exec -it deployment/seafile -n seafile -- nc -zv mariadb 3306

# 或使用 telnet
kubectl exec -it deployment/seafile -n seafile -- telnet mariadb 3306
```

#### 步骤 4: 检查数据库密码

```bash
# 查看 Secret 中的密码
kubectl get secret seafile-secrets -n seafile -o jsonpath='{.data.mysql-root-password}' | base64 -d
echo ""

# 检查 Seafile Deployment 中的环境变量
kubectl get deployment seafile -n seafile -o yaml | grep -A 5 "env:"

# 确保密码匹配
# MariaDB 使用的密码
kubectl get deployment mariadb -n seafile -o yaml | grep -A 5 "MYSQL_ROOT_PASSWORD"

# Seafile 使用的密码
kubectl get deployment seafile -n seafile -o yaml | grep -A 5 "DB_ROOT_PASSWD"
```

#### 步骤 5: 进入 MariaDB 容器检查

```bash
# 进入 MariaDB 容器
kubectl exec -it deployment/mariadb -n seafile -- bash

# 登录 MySQL
mysql -u root -p
# 输入密码: db_root_password

# 检查数据库
SHOW DATABASES;

# 检查用户权限
SELECT User, Host FROM mysql.user;

# 创建 Seafile 需要的数据库（如果不存在）
CREATE DATABASE IF NOT EXISTS ccnet CHARACTER SET utf8;
CREATE DATABASE IF NOT EXISTS seafile CHARACTER SET utf8;
CREATE DATABASE IF NOT EXISTS seahub CHARACTER SET utf8;

# 退出
EXIT;
exit
```

### 2.2 数据库初始化失败

**症状**: MariaDB 容器反复重启

```bash
# 查看 MariaDB 日志
kubectl logs -f deployment/mariadb -n seafile

# 检查数据目录
kubectl exec -it deployment/mariadb -n seafile -- ls -la /var/lib/mysql

# 如果数据目录损坏，需要清空重新初始化
# 警告：这会删除所有数据！
ssh k8s-node1
sudo rm -rf /data/mariadb/*
# 然后重启 MariaDB Pod
kubectl rollout restart deployment/mariadb -n seafile
```

---

## 3. Seafile 无法访问问题

### 3.1 无法通过 NodePort 访问

**症状**: 浏览器无法打开 http://172.22.175.252:30080

**排查步骤**:

#### 步骤 1: 检查 Seafile Pod 状态

```bash
# 检查 Pod 是否运行
kubectl get pods -n seafile -l app=seafile

# 检查 Pod 日志
kubectl logs -f deployment/seafile -n seafile
```

#### 步骤 2: 检查 Service 配置

```bash
# 检查 Service
kubectl get svc seafile -n seafile

# 应该看到类似输出：
# NAME      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
# seafile   NodePort   10.96.x.x      <none>        80:30080/TCP   10m

# 检查 Service 详情
kubectl describe svc seafile -n seafile

# 确认 Endpoints 存在
kubectl get endpoints seafile -n seafile
```

#### 步骤 3: 测试从集群内访问

```bash
# 从集群内测试访问 Service
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -n seafile -- curl -I http://seafile

# 测试访问 Pod IP
kubectl get pods -n seafile -l app=seafile -o wide
# 记录 Pod IP，然后测试
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -n seafile -- curl -I http://<pod-ip>
```

#### 步骤 4: 检查防火墙规则

```bash
# 在所有节点上检查防火墙
ssh k8s-master
sudo iptables -L -n | grep 30080
sudo firewall-cmd --list-ports  # 如果使用 firewalld

# 如果端口被阻止，需要开放
sudo firewall-cmd --permanent --add-port=30080/tcp
sudo firewall-cmd --reload

# 或者 (iptables)
sudo iptables -A INPUT -p tcp --dport 30080 -j ACCEPT
```

#### 步骤 5: 检查 kube-proxy

```bash
# 检查 kube-proxy 是否运行
kubectl get pods -n kube-system | grep kube-proxy

# 查看 kube-proxy 日志
kubectl logs -n kube-system <kube-proxy-pod-name>

# 检查 iptables 规则
sudo iptables -t nat -L -n | grep 30080
```

### 3.2 访问缓慢或超时

**可能原因**:
- 资源不足
- 网络延迟
- Memcached 未运行
- 数据库查询慢

**排查方法**:

```bash
# 检查资源使用情况
kubectl top pods -n seafile
kubectl top nodes

# 检查 Memcached 状态
kubectl get pods -n seafile -l app=memcached
kubectl logs -f deployment/memcached -n seafile

# 检查 Seafile 日志中的性能问题
kubectl logs -f deployment/seafile -n seafile | grep -i "slow\|timeout\|error"

# 进入 Seafile 容器检查
kubectl exec -it deployment/seafile -n seafile -- bash

# 测试 Memcached 连接
telnet memcached 11211
# 输入: stats
# 应该看到 Memcached 统计信息
```

### 3.3 页面显示错误

**症状**: 页面显示 500 错误或无法加载

```bash
# 查看 Seafile 日志
kubectl logs -f deployment/seafile -n seafile

# 进入容器检查配置
kubectl exec -it deployment/seafile -n seafile -- bash

# 查看 Seafile 配置文件
cat /shared/seafile/conf/ccnet.conf
cat /shared/seafile/conf/seafile.conf
cat /shared/seafile/conf/seahub_settings.py

# 检查 SEAFILE_SERVER_HOSTNAME 配置
# 应该匹配访问的 IP 地址
echo $SEAFILE_SERVER_HOSTNAME
```

---

## 4. 存储权限问题

### 4.1 权限被拒绝错误

**症状**: Pod 日志显示 `Permission denied` 错误

```bash
# 查看具体错误
kubectl logs deployment/mariadb -n seafile | grep -i "permission"
kubectl logs deployment/seafile -n seafile | grep -i "permission"
```

**解决方案**:

#### 对于 MariaDB 存储

```bash
# 在 k8s-node1 上执行
ssh k8s-node1

# 检查目录权限
ls -la /data/mariadb

# 修改权限
sudo chown -R 999:999 /data/mariadb
# 或者
sudo chmod 777 /data/mariadb

# 如果目录不存在，创建它
sudo mkdir -p /data/mariadb
sudo chmod 777 /data/mariadb
```

#### 对于 Seafile 存储

```bash
# 在 k8s-node2 上执行
ssh k8s-node2

# 检查目录权限
ls -la /data/seafile

# 修改权限
sudo chown -R 8000:8000 /data/seafile
# 或者
sudo chmod 777 /data/seafile

# 如果目录不存在，创建它
sudo mkdir -p /data/seafile
sudo chmod 777 /data/seafile
```

### 4.2 SELinux 问题

如果启用了 SELinux，可能会阻止 Pod 访问本地存储：

```bash
# 检查 SELinux 状态
sestatus

# 临时禁用 SELinux（测试用）
sudo setenforce 0

# 永久禁用（不推荐）
# 编辑 /etc/selinux/config
SELINUX=disabled

# 或者配置 SELinux 策略允许访问
sudo chcon -Rt svirt_sandbox_file_t /data/mariadb
sudo chcon -Rt svirt_sandbox_file_t /data/seafile
```

### 4.3 磁盘空间不足

```bash
# 检查磁盘使用情况
df -h /data/mariadb
df -h /data/seafile

# 在节点上清理空间
# k8s-node1
ssh k8s-node1
sudo du -sh /data/mariadb/*
# 根据需要清理旧数据或扩展磁盘

# k8s-node2
ssh k8s-node2
sudo du -sh /data/seafile/*
```

---

## 5. 日志查看方法

### 5.1 实时查看日志

```bash
# 实时查看 MariaDB 日志
kubectl logs -f deployment/mariadb -n seafile

# 实时查看 Memcached 日志
kubectl logs -f deployment/memcached -n seafile

# 实时查看 Seafile 日志
kubectl logs -f deployment/seafile -n seafile

# 查看最近 100 行日志
kubectl logs --tail=100 deployment/seafile -n seafile

# 查看过去 1 小时的日志
kubectl logs --since=1h deployment/seafile -n seafile
```

### 5.2 查看之前的日志

```bash
# 查看 Pod 重启前的日志（如果 Pod 重启了）
kubectl logs <pod-name> -n seafile --previous

# 或
kubectl logs deployment/seafile -n seafile --previous
```

### 5.3 导出日志到文件

```bash
# 导出 Seafile 日志
kubectl logs deployment/seafile -n seafile > seafile.log

# 导出所有 Pod 的日志
for pod in $(kubectl get pods -n seafile -o name); do
  kubectl logs $pod -n seafile > $(basename $pod).log
done
```

### 5.4 查看特定容器的日志（多容器 Pod）

```bash
# 如果 Pod 有多个容器，指定容器名
kubectl logs <pod-name> -c <container-name> -n seafile

# 列出 Pod 的所有容器
kubectl get pod <pod-name> -n seafile -o jsonpath='{.spec.containers[*].name}'
```

### 5.5 使用 stern 工具（推荐）

stern 可以同时查看多个 Pod 的日志：

```bash
# 安装 stern
wget https://github.com/stern/stern/releases/download/v1.22.0/stern_1.22.0_linux_amd64.tar.gz
tar -xzf stern_1.22.0_linux_amd64.tar.gz
sudo mv stern /usr/local/bin/

# 查看 seafile 命名空间的所有日志
stern -n seafile .

# 只查看 seafile 相关的 Pod
stern -n seafile seafile

# 查看最近 5 分钟的日志
stern -n seafile . --since=5m
```

---

## 6. 常用调试命令

### 6.1 资源状态检查

```bash
# 查看所有资源
kubectl get all -n seafile

# 查看资源使用情况
kubectl top nodes
kubectl top pods -n seafile

# 查看事件（最近发生的事情）
kubectl get events -n seafile --sort-by='.lastTimestamp'

# 查看集群信息
kubectl cluster-info
kubectl get nodes -o wide
```

### 6.2 网络调试

```bash
# 创建调试 Pod
kubectl run debug --image=nicolaka/netshoot -it --rm --restart=Never -n seafile -- bash

# 在调试 Pod 中：
# DNS 查询
nslookup mariadb
nslookup seafile
nslookup memcached

# 测试端口连通性
nc -zv mariadb 3306
nc -zv memcached 11211
nc -zv seafile 80

# 跟踪路由
traceroute mariadb

# 抓包
tcpdump -i any port 3306

# curl 测试
curl -v http://seafile
```

### 6.3 容器内调试

```bash
# 进入 Seafile 容器
kubectl exec -it deployment/seafile -n seafile -- bash

# 在容器内执行命令（不进入）
kubectl exec deployment/seafile -n seafile -- ls -la /shared
kubectl exec deployment/seafile -n seafile -- cat /shared/seafile/conf/seafile.conf

# 从 Pod 复制文件到本地
kubectl cp seafile/<pod-name>:/shared/seafile/conf/seafile.conf ./seafile.conf -n seafile

# 从本地复制文件到 Pod
kubectl cp ./seafile.conf seafile/<pod-name>:/shared/seafile/conf/seafile.conf -n seafile
```

### 6.4 配置查看

```bash
# 查看 Deployment 配置
kubectl get deployment seafile -n seafile -o yaml
kubectl get deployment mariadb -n seafile -o yaml
kubectl get deployment memcached -n seafile -o yaml

# 查看 Service 配置
kubectl get svc -n seafile -o yaml

# 查看 PV/PVC 配置
kubectl get pv -o yaml
kubectl get pvc -n seafile -o yaml

# 查看 Secret
kubectl get secret seafile-secrets -n seafile -o yaml
kubectl get secret seafile-secrets -n seafile -o jsonpath='{.data}' | jq
```

### 6.5 强制删除问题资源

```bash
# 如果 Pod 一直处于 Terminating 状态
kubectl delete pod <pod-name> -n seafile --force --grace-period=0

# 如果 PVC 无法删除（有 finalizer）
kubectl patch pvc <pvc-name> -n seafile -p '{"metadata":{"finalizers":null}}'

# 如果 Namespace 无法删除
kubectl get namespace seafile -o json | jq '.spec.finalizers = []' | kubectl replace --raw /api/v1/namespaces/seafile/finalize -f -
```

### 6.6 备份和恢复

```bash
# 备份配置
kubectl get all -n seafile -o yaml > seafile-backup.yaml

# 备份数据（需要在节点上执行）
# MariaDB 数据
ssh k8s-node1
sudo tar -czf /tmp/mariadb-backup-$(date +%Y%m%d).tar.gz /data/mariadb

# Seafile 数据
ssh k8s-node2
sudo tar -czf /tmp/seafile-backup-$(date +%Y%m%d).tar.gz /data/seafile

# 恢复数据
# 1. 删除现有 Deployment
kubectl delete deployment seafile mariadb memcached -n seafile

# 2. 在节点上恢复数据
ssh k8s-node1
sudo rm -rf /data/mariadb/*
sudo tar -xzf /tmp/mariadb-backup-20231201.tar.gz -C /

# 3. 重新部署
kubectl apply -f manifests/
```

---

## 7. 问题排查流程图

```
问题：Seafile 无法访问
        │
        ├─> 检查 Seafile Pod 状态
        │   ├─> Pending → 检查 PVC/资源/节点
        │   ├─> CrashLoopBackOff → 检查日志/数据库/存储
        │   └─> Running → 继续下一步
        │
        ├─> 检查 Service 配置
        │   ├─> Service 不存在 → 创建 Service
        │   ├─> Endpoints 为空 → 检查 Pod 标签
        │   └─> 正常 → 继续下一步
        │
        ├─> 测试从集群内访问
        │   ├─> 可以访问 → 检查 NodePort/防火墙
        │   └─> 无法访问 → 检查网络/DNS
        │
        └─> 测试从集群外访问
            ├─> 可以访问 → 问题解决
            └─> 无法访问 → 检查防火墙/kube-proxy
```

---

## 8. 获取帮助

如果以上方法都无法解决问题：

1. **收集信息**:
   ```bash
   # 生成诊断报告
   kubectl get all -n seafile -o wide > diagnosis.txt
   kubectl describe pods -n seafile >> diagnosis.txt
   kubectl logs deployment/seafile -n seafile --tail=200 >> diagnosis.txt
   kubectl logs deployment/mariadb -n seafile --tail=200 >> diagnosis.txt
   kubectl get events -n seafile --sort-by='.lastTimestamp' >> diagnosis.txt
   ```

2. **查看官方文档**:
   - Kubernetes 官方文档: https://kubernetes.io/docs/
   - Seafile 官方文档: https://manual.seafile.com/

3. **社区支持**:
   - Kubernetes 社区论坛
   - Seafile 论坛
   - Stack Overflow

4. **提交 Issue**:
   - 包含完整的错误信息
   - 提供环境信息（Kubernetes 版本、节点配置等）
   - 附上诊断报告

---

## 9. 预防性维护

为了避免问题发生，建议：

1. **定期备份**: 每天备份数据库和文件
2. **监控资源**: 使用 Prometheus + Grafana 监控
3. **日志收集**: 配置集中式日志系统
4. **定期更新**: 及时更新镜像和 Kubernetes 版本
5. **测试环境**: 先在测试环境验证变更
6. **文档记录**: 记录所有配置变更和问题解决过程

---

## 10. 应急联系

生产环境出现严重问题时的应急措施：

```bash
# 1. 立即回滚到上一个版本
kubectl rollout undo deployment/seafile -n seafile

# 2. 扩容资源
kubectl scale deployment/memcached --replicas=2 -n seafile

# 3. 重启服务
kubectl rollout restart deployment/seafile -n seafile

# 4. 临时禁用部分功能
# 修改 Seafile 配置，禁用非关键功能

# 5. 切换到备份实例（如果有）
# 修改 Service 指向备份 Pod
```