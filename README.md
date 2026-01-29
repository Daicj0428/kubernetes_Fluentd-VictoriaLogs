# VictoriaLogs 日志系统部署方案

基于 Kubernetes 的企业级日志收集与存储解决方案，采用 **Fluentd + VictoriaLogs** 架构。

## 📋 目录

- [架构概览](#架构概览)
- [组件说明](#组件说明)
- [环境要求](#环境要求)
- [快速开始](#快速开始)
- [文件说明](#文件说明)
- [部署前准备](#部署前准备)
- [部署步骤](#部署步骤)
- [验证部署](#验证部署)
- [运维指南](#运维指南)
- [故障排查](#故障排查)
- [性能优化](#性能优化)
- [安全加固](#安全加固)

---

## 🏗️ 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                      Kubernetes Cluster                      │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │   Pod Logs   │    │ System Logs  │    │  Audit Logs  │   │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘   │
│         │                   │                   │           │
│         └───────────────────┼───────────────────┘           │
│                             ▼                               │
│              ┌──────────────────────────────┐                │
│              │  Fluentd DaemonSet          │                │
│              │  (每节点一个实例)             │                │
│              │  - 日志采集                   │                │
│              │  - 元数据丰富                 │                │
│              │  - 格式转换                   │                │
│              └──────────────┬───────────────┘                │
│                             │                                │
│                             ▼                                │
│              ┌──────────────────────────────┐                │
│              │  VictoriaLogs StatefulSet     │                │
│              │  - 日志存储                   │                │
│              │  - 高效查询                   │                │
│              │  - 自动保留                   │                │
│              └──────────────┬───────────────┘                │
│                             │                                │
│              ┌──────────────┴──────────────┐                │
│              │                             │                │
│         ┌────▼────┐                 ┌─────▼────┐            │
│         │  Query  │                 │  Metrics │            │
│         │  Web UI │                 │ Prometheus│            │
│         └─────────┘                 └──────────┘            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌────────────────┐
                    │  NFS Storage    │
                    │  (持久化存储)    │
                    └────────────────┘
```

---

## 📦 组件说明

| 组件 | 类型 | 功能 | 副本数 |
|------|------|------|--------|
| **NFS Provisioner** | Deployment | 动态提供 NFS 存储 | 1 |
| **VictoriaLogs** | StatefulSet | 日志存储与查询 | 1 (可扩展) |
| **Fluentd** | DaemonSet | 日志采集代理 | 每节点 1 个 |
| **Fluentd Service** | Service | 暴露 Fluentd 指标 | - |
| **VictoriaLogs Service** | Service | 内部访问端点 | - |
| **VictoriaLogs External** | Service | 外部访问端点 | - |
| **ServiceMonitors** | ServiceMonitor | Prometheus 监控 | - |
| **PrometheusRules** | PrometheusRule | 告警规则 | - |
| **NetworkPolicies** | NetworkPolicy | 网络隔离策略 | - |
| **HPA** | HPA | 自动水平扩容 | - |

---

## ⚙️ 环境要求

### Kubernetes 集群
- **版本**: 1.20+
- **节点数**: 至少 3 个（生产环境推荐）
- **节点资源**:
  - Control Plane: 2 CPU / 4GB RAM
  - Worker Nodes: 4 CPU / 8GB RAM / 100GB 磁盘

### 存储要求
- **NFS 服务器**: 可用并配置正确
- **存储空间**: 至少 200GB (根据日志量调整)
- **IOPS**: 建议 1000+ (NFS 可能成为瓶颈)

### 网络要求
- Pod 网络互通 (默认 K8s 网络)
- 可访问外部镜像仓库
- 可选: 外部访问 NodePort (30428, 30429)

### 软件依赖
- kubectl 1.20+
- 可选: Helm 3.x (用于部署 Prometheus Operator)
- 可选: Velero (用于备份)

---

## 🚀 快速开始

### 一键部署 (开发/测试环境)

```bash
# 1. 克隆仓库
cd victorailogs

# 2. 修改 NFS 配置 (必须!)
# 编辑 01-nfs-storage.yaml，替换 NFS_SERVER 和 NFS_PATH

# 3. 按顺序部署
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-nfs-storage.yaml
kubectl apply -f 02-victorialogs-core.yaml
kubectl apply -f 03-fluentd-daemonset.yaml
kubectl apply -f 04-fluentd-service.yaml

# 4. 验证
kubectl get pods -n kube-logging
```

### 生产环境部署

```bash
# 完整部署（包含监控、安全策略等）
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-nfs-storage.yaml
kubectl apply -f 02-victorialogs-core.yaml
kubectl apply -f 03-fluentd-daemonset.yaml
kubectl apply -f 04-fluentd-service.yaml
kubectl apply -f 05-monitor.yaml           # 需要 Prometheus Operator
kubectl apply -f 06-network-policy.yaml
kubectl apply -f 07-hpa.yaml
kubectl apply -f 08-pvc-monitoring.yaml
```

---

## 📁 文件说明

| 文件 | 说明 | 必需 |
|------|------|------|
| `00-namespace.yaml` | 创建 `kube-logging` 命名空间 | ✅ |
| `01-nfs-storage.yaml` | NFS 存储供应器配置 | ✅ |
| `02-victorialogs-core.yaml` | VictoriaLogs 核心组件 | ✅ |
| `03-fluentd-daemonset.yaml` | Fluentd 日志采集器 | ✅ |
| `04-fluentd-service.yaml` | Fluentd 服务暴露 | ✅ |
| `05-monitor.yaml` | Prometheus 监控配置 | ⚠️ |
| `06-network-policy.yaml` | 网络安全策略 | ⚠️ |
| `07-hpa.yaml` | 自动水平扩容 | ⚠️ |
| `08-pvc-monitoring.yaml` | 存储监控告警 | ⚠️ |

**说明**:
- ✅ = 核心组件，必须部署
- ⚠️ = 推荐部署，增强功能
- ❌ = 参考文档，无需部署

---

## 🔧 部署前准备

### 1. 修改 NFS 配置 (必须!)

编辑 `01-nfs-storage.yaml`:

```yaml
env:
  - name: NFS_SERVER
    value: <你的NFS服务器IP>      # 例如: 192.168.1.100
  - name: NFS_PATH
    value: <你的NFS共享路径>      # 例如: /data/k8s-logs

volumes:
  - name: nfs-client-root
    nfs:
      server: <你的NFS服务器IP>      # 同上
      path: <你的NFS共享路径>      # 同上
```

### 2. 准备 NFS 服务器

```bash
# 在 NFS 服务器上创建共享目录
sudo mkdir -p /data/k8s-logs
sudo chmod 777 /data/k8s-logs

# 配置 NFS 导出
# 编辑 /etc/exports
/data/k8s-logs *(rw,sync,no_subtree_check,no_root_squash)

# 应用配置
sudo exportfs -ra
sudo systemctl restart nfs-server

# 验证
showmount -e localhost
```

### 3. 检查镜像仓库

确保以下镜像可访问:
- `dai30.test.com/k8s/nfs-subdir-external-provisione:v4.0.0`
- `victoriametrics/victoria-logs:latest`
- `dai30.test.com/k8s/fluentd:v3.1.0`

如需使用公共镜像，请相应修改镜像地址。

### 4. 安装 Prometheus Operator (可选)

如果需要监控功能，先安装 Prometheus Operator:

```bash
# 使用 Helm 安装
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

---

## 📝 部署步骤

### 步骤 1: 创建命名空间

```bash
kubectl apply -f 00-namespace.yaml
```

验证:
```bash
kubectl get namespace kube-logging
```

### 步骤 2: 部署 NFS 存储供应器

```bash
kubectl apply -f 01-nfs-storage.yaml
```

验证:
```bash
# 检查 Pod
kubectl get pods -n kube-system -l app=nfs-client-provisioner

# 检查 StorageClass
kubectl get storageclass managed-nfs-storage
```

等待 Pod 变为 `Running` 状态。

### 步骤 3: 部署 VictoriaLogs

```bash
kubectl apply -f 02-victorialogs-core.yaml
```

验证:
```bash
# 检查 Pod
kubectl get pods -n kube-logging -l app=victoria-logs

# 检查 PVC
kubectl get pvc -n kube-logging

# 检查服务
kubectl get svc -n kube-logging
```

等待 Pod 变为 `Running` 且 PVC 状态为 `Bound`。

### 步骤 4: 部署 Fluentd

```bash
kubectl apply -f 03-fluentd-daemonset.yaml
kubectl apply -f 04-fluentd-service.yaml
```

验证:
```bash
# 检查 DaemonSet
kubectl get ds -n kube-logging fluentd

# 检查 Pods (应该在每个节点运行一个)
kubectl get pods -n kube-logging -l app=fluentd

# 查看日志
kubectl logs -n kube-logging -l app=fluentd --tail=50
```

### 步骤 5: 部署监控组件 (可选)

```bash
kubectl apply -f 05-monitor.yaml
kubectl apply -f 08-pvc-monitoring.yaml
```

验证:
```bash
# 检查 ServiceMonitors
kubectl get servicemonitor -n kube-logging

# 检查 PrometheusRules
kubectl get prometheusrules -n kube-logging

# 在 Prometheus UI 中查看目标状态
# kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090
```

### 步骤 6: 部署安全策略 (推荐)

```bash
kubectl apply -f 06-network-policy.yaml
```

验证:
```bash
kubectl get networkpolicy -n kube-logging
```

### 步骤 7: 部署自动扩容 (可选)

```bash
kubectl apply -f 07-hpa.yaml
```

验证:
```bash
kubectl get hpa -n kube-logging
```

---

## ✅ 验证部署

### 1. 检查所有组件状态

```bash
# 检查所有 Pod
kubectl get pods -n kube-logging

# 期望输出:
# NAME                              READY   STATUS    RESTARTS   AGE
# victoria-logs-0                   1/1     Running   0          5m
# fluentd-xxxxx                     1/1     Running   0          3m
# fluentd-yyyyy                     1/1     Running   0          3m
# ...

# 检查服务
kubectl get svc -n kube-logging

# 检查 PVC
kubectl get pvc -n kube-logging
```

### 2. 测试日志采集

```bash
# 创建测试 Pod 生成日志
kubectl run log-generator --image=busybox --restart=Never -n kube-logging \
  -- sh -c 'for i in $(seq 1 100); do echo "Test log message $i"; sleep 2; done'

# 查看 Fluentd 日志 (应该看到日志被采集)
kubectl logs -n kube-logging -l app=fluentd --tail=20 | grep -i "Test log"
```

### 3. 访问 VictoriaLogs Web UI

```bash
# 方式 1: 端口转发 (推荐)
kubectl port-forward -n kube-logging svc/victoria-logs 9428:9428

# 浏览器访问: http://localhost:9428

# 方式 2: 使用 NodePort 服务
kubectl get svc victoria-logs-external -n kube-logging
# 浏览器访问: http://<节点IP>:30428
```

在 VictoriaLogs UI 中执行查询:
```
_msg: "Test log"
```

### 4. 检查监控指标

```bash
# Fluentd 指标
curl http://localhost:24231/metrics

# VictoriaLogs 指标
kubectl port-forward -n kube-logging svc/victoria-logs 9428:9428
curl http://localhost:9428/metrics
```

### 5. 验证日志流

```bash
# 查看 Fluentd 输出插件状态
kubectl exec -n kube-logging -l app=fluentd -- \
  curl -s http://localhost:24231/api/plugins.json | jq '.plugins[] | select(.type=="http")'

# 查看 VictoriaLogs 摄入速率
kubectl exec -n kube-logging victoria-logs-0 -- \
  curl -s http://localhost:9428/api/v1/status
```

---

## 🔍 运维指南

### 日常维护

#### 查看 VictoriaLogs 存储使用情况

```bash
# 访问 Web UI 的 /api/v1/status 端点
kubectl exec -n kube-logging victoria-logs-0 -- \
  curl -s http://localhost:9428/api/v1/status | jq
```

#### 清理旧日志

VictoriaLogs 自动根据保留策略清理日志，默认 30 天。如需修改，编辑 `02-victorialogs-core.yaml`:

```yaml
args:
  - "--retentionPeriod=90d"  # 修改为 90 天
```

然后重启 Pod:

```bash
kubectl rollout restart statefulset victoria-logs -n kube-logging
```

#### 调整存储大小

如果需要扩容 PVC (存储类支持扩容):

```bash
# 编辑 PVC
kubectl edit pvc storage-victoria-logs-0 -n kube-logging

# 修改 storage 字段
# storage: 200Gi

# 等待扩容完成
kubectl get pvc -n kube-logging -w
```

### 备份与恢复

#### 使用 Velero 备份

```bash
# 安装 Velero (如未安装)
# https://velero.io/docs/

# 创建备份
velero backup create victoria-logs-backup \
  --include-namespaces=kube-logging \
  --storage-location=default \
  --wait

# 定期备份 (每天凌晨 2 点)
velero schedule create daily-logs-backup \
  --schedule="0 2 * * *" \
  --include-namespaces=kube-logging \
  --storage-location=default \
  --ttl=720h  # 保留 30 天
```

#### 恢复备份

```bash
# 列出所有备份
velero backup get

# 恢复
velero restore create --from-backup victoria-logs-backup
```

### 日志查询

#### 基本查询语法

VictoriaLogs 使用类似 Grafana Loki 的查询语法:

```
# 查询特定命名空间的日志
{k8s_namespace="default"}

# 查询特定 Pod 的日志
{k8s_pod="my-app-*"}

# 查询特定容器
{k8s_container="backend"}

# 查询日志级别
log_level="ERROR"

# 组合查询
{k8s_namespace="default"} |= "error"

# 正则表达式
{k8s_pod=~".*-.*"} |~ "Exception"

# 时间范围
{k8s_namespace="default"}[5m]  # 最近 5 分钟
```

#### 常用查询示例

```bash
# 查询所有错误日志
log_level="ERROR"

# 查询特定应用的错误
{stream_app="myapp"} |= "error"

# 查询慢请求
| duration_ms > 1000

# 查询 HTTP 5xx 错误
{stream_app="nginx"} |~ "5[0-9]{2}"

# 查询特定时间段的日志
{stream_k8s_namespace="kube-system"} "2024-01-01T00:00:00Z"-"2024-01-02T00:00:00Z"
```

---

## 🐛 故障排查

### 问题 1: VictoriaLogs Pod 无法启动

**症状**: Pod 处于 `CrashLoopBackOff` 状态

**排查步骤**:

```bash
# 1. 查看 Pod 状态
kubectl describe pod -n kube-logging -l app=victoria-logs

# 2. 查看日志
kubectl logs -n kube-logging -l app=victoria-logs

# 3. 检查 PVC
kubectl get pvc -n kube-logging
kubectl describe pvc -n kube-logging

# 4. 检查 NFS 连接
kubectl exec -n kube-logging victoria-logs-0 -- df -h
```

**常见原因**:
- NFS 服务器不可访问
- PVC 无法绑定
- 存储权限问题
- 资源不足

**解决方法**:
```bash
# 重新部署 PVC
kubectl delete pvc storage-victoria-logs-0 -n kube-logging
kubectl delete pod victoria-logs-0 -n kube-logging
# PVC 会自动重建
```

### 问题 2: Fluentd 无法发送日志

**症状**: 日志堆积在 buffer 中，VictoriaLogs 无数据

**排查步骤**:

```bash
# 1. 查看 Fluentd 日志
kubectl logs -n kube-logging -l app=fluentd | grep -i error

# 2. 检查网络连接
kubectl exec -n kube-logging -l app=fluentd -- \
  curl -v http://victoria-logs.kube-logging.svc.cluster.local:9428/health

# 3. 查看 buffer 状态
kubectl exec -n kube-logging -l app=fluentd -- \
  curl -s http://localhost:24231/api/buffers.json | jq

# 4. 查看 VictoriaLogs 摄入指标
kubectl port-forward -n kube-logging svc/victoria-logs 9428:9428
curl http://localhost:9428/metrics | grep vl_ingested_rows_total
```

**常见原因**:
- VictoriaLogs 服务不可用
- 网络策略阻止
- Buffer 满了
- 日志格式不匹配

**解决方法**:
```bash
# 检查网络策略
kubectl get networkpolicy -n kube-logging

# 如果网络策略阻止，修改策略或临时删除
kubectl delete networkpolicy victoria-logs-netpol -n kube-logging

# 重启 Fluentd
kubectl rollout restart daemonset fluentd -n kube-logging
```

### 问题 3: PVC 无法绑定

**症状**: PVC 状态一直是 `Pending`

**排查步骤**:

```bash
# 1. 查看 PVC 详情
kubectl describe pvc -n kube-logging

# 2. 检查 StorageClass
kubectl get storageclass

# 3. 检查 NFS Provisioner
kubectl get pods -n kube-system -l app=nfs-client-provisioner
kubectl logs -n kube-system -l app=nfs-client-provisioner

# 4. 测试 NFS 连接
# 在任意节点上执行
showmount -e <NFS_SERVER_IP>
```

**常见原因**:
- NFS 服务器配置错误
- StorageClass 不存在
- Provisioner Pod 异常
- NFS 导出路径错误

**解决方法**:
```bash
# 检查 NFS 服务器配置
sudo exportfs -v

# 重启 NFS Provisioner
kubectl delete pod -n kube-system -l app=nfs-client-provisioner

# 验证 StorageClass
kubectl get sc managed-nfs-storage -o yaml
```

### 问题 4: 磁盘空间不足

**症状**: Pod 因磁盘空间不足被驱逐

**排查步骤**:

```bash
# 1. 检查 PVC 使用情况
kubectl exec -n kube-logging victoria-logs-0 -- df -h /storage

# 2. 检查节点磁盘
kubectl describe nodes | grep -A 5 "Allocated resources"

# 3. 查看 VictoriaLogs 存储使用
kubectl port-forward -n kube-logging svc/victoria-logs 9428:9428
curl http://localhost:9428/api/v1/status | jq '.dataSizeBytes'
```

**解决方法**:
```bash
# 调整保留策略 (缩短保留时间)
kubectl edit statefulset victoria-logs -n kube-logging
# 修改 --retentionPeriod 参数

# 扩容 PVC (存储类支持扩容)
kubectl edit pvc storage-victoria-logs-0 -n kube-logging
# 增加 storage 值
```

### 问题 5: 查询性能慢

**症状**: VictoriaLogs 查询响应时间长

**排查步骤**:

```bash
# 1. 检查查询指标
curl http://localhost:9428/metrics | grep vl_search_

# 2. 检查系统资源
kubectl top pod -n kube-logging victoria-logs-0

# 3. 查看 VictoriaLogs 日志
kubectl logs -n kube-logging victoria-logs-0 | grep -i "slow query"
```

**优化建议**:
```bash
# 1. 优化查询条件
# 使用具体的 stream 标签而非全文搜索
{stream_k8s_namespace="default"} |= "error"  # 好
| "error"  # 不好

# 2. 增加 VictoriaLogs 资源
kubectl edit statefulset victoria-logs -n kube-logging
# 增加 memory 和 cpu limits

# 3. 缩短时间范围
# 避免查询过长时间范围的数据
```

---

## ⚡ 性能优化

### 1. VictoriaLogs 调优

#### 根据集群规模调整资源

**小集群 (<100 Pods)**:
```yaml
resources:
  requests:
    memory: 1Gi
    cpu: 500m
  limits:
    memory: 2Gi
    cpu: 1000m
```

**中集群 (100-500 Pods)**:
```yaml
resources:
  requests:
    memory: 2Gi
    cpu: 1000m
  limits:
    memory: 4Gi
    cpu: 2000m
```

**大集群 (>500 Pods)**:
```yaml
# 考虑多副本部署
spec:
  replicas: 3
  # 配置负载均衡
```

#### 启用查询缓存

```yaml
args:
  - "--cacheExpireDuration=5m"
  - "--cacheSizeBytes=100000000"  # 100MB
```

#### 调整摄入参数

```yaml
args:
  - "--insert.maxQueueSizeBytes=1GB"    # 增大队列
  - "--insert.maxConcurrentInserts=100"  # 并发写入
  - "--memory.allowedPercent=80"         # 内存使用上限
```

### 2. Fluentd 调优

#### 调整 Worker 数量

```yaml
<system>
  workers 4  # 根据 CPU 核心数调整
</system>
```

#### 优化 Buffer 配置

```yaml
<buffer stream_k8s_namespace,stream_k8s_pod,stream_k8s_container>
  @type file
  path /var/log/fluentd-buffer/container
  flush_mode interval
  flush_interval 3s        # 更频繁刷新
  flush_thread_count 4     # 增加线程数
  chunk_limit_size 64M     # 增大块大小
  total_limit_size 50G     # 增大总缓冲
</buffer>
```

#### 启用日志采样 (生产环境)

```yaml
# 对 DEBUG 日志进行采样
<filter **>
  @type sampling
  rate 0.1  # 保留 10%
  <regexp>
    key log_level
    pattern /^DEBUG$/
  </regexp>
</filter>
```

### 3. 存储优化

#### 使用高性能存储

生产环境建议使用 Local PV 或高性能 SSD:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-ssd
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

#### 启用压缩

VictoriaLogs 默认启用压缩，无需额外配置。

### 4. 网络优化

#### 使用主机网络 (可选)

```yaml
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
```

#### 调整 TCP 参数

```yaml
env:
  - name: TCP_FASTOPEN
    value: "3"
```

---

## 🔒 安全加固

### 1. 启用认证

VictoriaLogs 支持基于 HTTP 头的认证:

```yaml
args:
  - "--auth.token=${VL_AUTH_TOKEN}"

env:
  - name: VL_AUTH_TOKEN
    valueFrom:
      secretKeyRef:
        name: victoria-logs-auth
        key: token
```

Fluentd 配置:

```yaml
headers:
  Authorization: "Bearer ${VL_AUTH_TOKEN}"
```

### 2. 启用 TLS 加密

```yaml
args:
  - "--tlsCertFile=/etc/tls/cert.pem"
  - "--tlsKeyFile=/etc/tls/key.pem"
  - "--httpsListenAddr=:9428"

volumeMounts:
  - name: tls-certs
    mountPath: /etc/tls
    readOnly: true
```

### 3. 网络隔离

已包含在 `06-network-policy.yaml` 中，限制 Pod 间通信。

### 4. RBAC 最小权限

当前 RBAC 配置已遵循最小权限原则，无需额外修改。

### 5. 容器安全

```yaml
securityContext:
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

### 6. 定期更新镜像

```bash
# 定期检查镜像更新
kubectl get pods -n kube-logging -o jsonpath='{.items[*].spec.containers[*].image}'

# 更新镜像
kubectl set image statefulset/victoria-logs victoria-logs=victoriametrics/victoria-logs:<new-tag> -n kube-logging
```

---

## 📊 监控告警

### 关键指标

| 指标 | 说明 | 告警阈值 |
|------|------|----------|
| `vl_ingested_rows_total` | 日志摄入速率 | > 50000/s |
| `process_resident_memory_bytes` | 内存使用 | > 75% |
| `vl_storage_size_bytes` | 存储使用 | > 80% |
| `fluentd_output_status_num_errors` | 错误率 | > 5/min |
| `up{job="victoria-logs"}` | 服务可用性 | = 0 |

### Prometheus 查询示例

```promql
# 每秒摄入日志数
rate(vl_ingested_rows_total[5m])

# 存储使用百分比
(vl_storage_size_bytes / vl_storage_max_disk_usage_bytes) * 100

# 查询延迟
rate(vl_search_request_duration_seconds_sum[5m]) / rate(vl_search_request_duration_seconds_count[5m])

# Fluentd buffer 使用率
fluentd_output_status_buffer_queue_length / 1000
```

### Grafana 仪表板

推荐安装 Grafana 并导入以下仪表板:

- VictoriaLogs 官方仪表板: https://grafana.com/grafana/dashboards/10229/
- Fluentd 监控仪表板: https://grafana.com/grafana/dashboards/10223/

---

## 📚 参考文档

- [VictoriaLogs 官方文档](https://docs.victoriametrics.com/victorialogs/)
- [Fluentd 官方文档](https://docs.fluentbit.io/manual/)
- [Kubernetes 日志架构](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [NFS 动态存储供应](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)

---

## 🤝 贡献

如有问题或建议，请提交 Issue 或 Pull Request。

---

## 📄 许可证

本项目仅供学习和参考使用。

---

## 📞 支持

如有部署或使用问题，请参考:
1. 本文档的 [故障排查](#故障排查) 章节
2. 各组件的官方文档
3. GitHub Issues

---
