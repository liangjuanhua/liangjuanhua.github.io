---
title: k8s故障案例
date: 2025-04-16 17:20:34
tags: 故障指南
categories: 故障指南
---
# **一、问题现象与背景**

某电商平台生产环境的Kubernetes集群在促销活动期间突发大规模Pod驱逐，具体表现如下：

1. **Pod频繁重启**：超过30%的Pod进入`Evicted`状态，核心服务（如订单支付、购物车）的Pod被反复驱逐。

2. **节点资源耗尽**：多个Worker节点的内存使用率超过95%，kubelet日志持续输出`MemoryPressure`警告。

3.  **监控告警**：
   Prometheus触发`node_memory_available_bytes < 10%`告警。
   Grafana面板显示部分节点的`kubelet_evictions`指标飙升。

4. **业务影响**：用户支付失败率从0.1%上升至15%，直接影响营收。

# **二、问题根因分析**

##### **1. 初步排查：节点与Pod状态**

```bash
# 查看节点资源使用情况（按内存排序）
kubectl top nodes --sort-by=memory

# 输出示例：
NAME           CPU(cores)  CPU%   MEMORY(bytes)  MEMORY%
worker-node-1  5800m       72%    6143Mi         98%
worker-node-2  4200m       52%    5892Mi         95%
worker-node-3  3800m       47%    4321Mi         70%

# 检查被驱逐的Pod
kubectl get pods -A -o wide | grep Evicted | wc -l  # 输出：47

# 查看某个被驱逐Pod的详细事件
kubectl describe pod payment-service-abcde -n production
```

**关键日志**：

```bash
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Warning  Evicted    2m    kubelet            The node was low on resource: memory.
  Normal   Killing    2m    kubelet            Stopping container payment-service
```

**结论**：节点内存不足触发kubelet的主动驱逐机制。

##### **2. 深入定位：资源消耗来源**

**步骤1：识别高内存消耗Pod**

```bash
# 按内存使用量排序所有Pod
kubectl top pods -A --sort-by=memory --use-protocol-buffers

# 输出示例：
NAMESPACE    POD_NAME                     MEMORY(Mi)
production   recommendation-service-xyz   1024
production   payment-service-abc          896
logging      fluentd-7k8jh                512
```

**发现**：`recommendation-service`的Pod内存占用异常高。

**步骤2：检查Pod资源限制配置**

```bash
kubectl get pod recommendation-service-xyz -n production -o yaml | grep -A 5 resources

# 输出示例：
resources:
  requests:
    cpu: "500m"
  limits:
    cpu: "1000m"
```

**问题**：该Pod未设置内存限制（`limits.memory`缺失），导致内存泄漏时无约束。

**步骤3：分析容器内存使用**

```bash
# 进入节点查看容器级内存占用（需SSH登录节点）
docker stats --format "table {{.Container}}\t{{.Name}}\t{{.MemUsage}}"

# 输出示例：
CONTAINER   NAME                      MEM USAGE
a1b2c3d4    recommendation-service    1.2GiB / 1.2GiB
```

**发现**：容器内存占用已突破1GiB，但未配置`limits.memory`，导致节点内存耗尽。

# **三、紧急处理措施**

##### **1. 快速扩容与负载分流**

- **横向扩展节点**：

```bash
# 使用Cluster Autoscaler自动扩容（假设配置了节点组）
kubectl scale deployment cluster-autoscaler --replicas=3 -n kube-system
```

• **临时调整Pod副本数**：

```bash
# 减少非核心服务副本数，释放资源
kubectl scale deployment batch-job-processor --replicas=0 -n background

# 增加核心服务副本数，分散负载
kubectl scale deployment payment-service --replicas=10 -n production
```

##### **2. 手动驱逐问题Pod**

```bash
# 强制删除高内存占用的Pod（触发重新调度）
kubectl delete pod recommendation-service-xyz -n production --force --grace-period=0

# 观察Pod重建后的内存使用
watch -n 1 "kubectl top pods -n production | grep recommendation-service"
```

##### **3. 动态调整kubelet驱逐阈值**

```bash
# 临时修改kubelet配置（避免更多Pod被驱逐）
sudo vi /etc/kubernetes/kubelet.conf
# 添加参数：
evictionHard:
  memory.available: "10%"
  nodefs.available: "5%"

# 重启kubelet
sudo systemctl restart kubelet
```

# **四、根因修复与长期优化**

##### **1. 资源配额规范化**

- **为所有Pod添加内存限制**：

```bash
# deployment.yaml示例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recommendation-service
spec:
  template:
    spec:
      containers:
      - name: app
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1024Mi"  # 硬性限制内存上限
            cpu: "2000m
```

• **启用命名空间级ResourceQuota**：

```bash
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.memory: "100Gi"
    limits.memory: "200Gi"
    pods: "200"
```

##### **2. 自动化弹性伸缩**

- **配置HPA（基于内存）**：

```bash
kubectl autoscale deployment recommendation-service -n production \
  --cpu-percent=70 \
  --memory-percent=80 \
  --min=3 \
  --max=20
```

• **使用VPA（垂直扩缩容）**：

```bash
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: recommendation-service-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: recommendation-service
  updatePolicy:
    updateMode: "Auto"
```

##### **3. 内存泄漏根治**

- **使用pprof进行堆分析**（以Go服务为例）：

```bash
import _ "net/http/pprof"

func main() {
  go func() {
    log.Println(http.ListenAndServe(":6060", nil))
  }()
  // 业务代码
}
```

```bash
# 生成堆内存快照
go tool pprof http://localhost:6060/debug/pprof/heap

# 分析内存分配
(pprof) top 10
(pprof) list leakFunction
```

- **优化代码逻辑**：修复循环引用、缓存未释放等问题。

# **五、监控与告警体系升级**

##### **1. Prometheus监控规则**

```bash
# prometheus-rules.yaml
groups:
- name: Kubernetes-Resource
  rules:
  - alert: NodeMemoryPressure
    expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "节点内存不足 ({{ $labels.instance }})"
      description: "节点 {{ $labels.instance }} 内存使用率超过85%，当前值 {{ $value }}%"

  - alert: PodEvictionRateHigh
    expr: rate(kube_pod_status_evicted[1h]) > 0
    for: 10m
    labels:
      severity: warning
```

##### **2. Grafana可视化面板**

-  **关键面板配置**：

- -  **节点资源视图**：`node_memory_available_bytes`、`node_cpu_usage`
  -  **Pod驱逐统计**：`sum(kube_pod_status_evicted) by (namespace)`
  -  **HPA伸缩历史**：`kube_horizontalpodautoscaler_status_current_replicas`

##### **3. 日志聚合分析**

- **Fluentd + Elasticsearch配置**：

```bash
<match kube.**>
  @type elasticsearch
  host elasticsearch.production.svc
  port 9200
  logstash_format true
  logstash_prefix k8s
</match>
```

• **关键日志筛选**：

```bash
# Kibana查询被驱逐Pod的日志
kubernetes.labels.app: "payment-service" AND message: "Evicted"
```

# **六、预防与容灾演练**

##### **1. 混沌工程实践**

-  **模拟节点故障**（使用Chaos Mesh）：

```bash
apiVersion: chaos-mesh.org/v1alpha1
kind: NodeFailure
metadata:
  name: node-failure-test
spec:
  action: shutdown
  duration: "10m"
  selector:
    nodes:
    - worker-node-1
```

- **验证集群自愈能力**：

-  观察Pod是否自动迁移到健康节点。
-  检查HPA是否按负载自动扩展。

##### **2. 定期压力测试**

-  **使用Locust模拟流量高峰**：

```bash
from locust import HttpUser, task

class PaymentUser(HttpUser):
    @task
    def create_order(self):
        self.client.post("/api/order", json={"items": [...]})
```

```bash
locust -f load_test.py --headless -u 1000 -r 100
```

##### **3. 架构优化**

- **服务网格化**：通过Istio实现熔断和降级。

```bash
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service.production.svc.cluster.local
  trafficPolicy:
    outlierDetection:
      consecutiveErrors: 5
      interval: 1m
      baseEjectionTime: 3m
```

# **七、总结与经验**

**解决效果**：

-  紧急措施在30分钟内恢复核心服务，Pod驱逐率降至0。
- 通过内存限制和HPA配置，集群资源利用率稳定在70%-80%。
- 后续3个月未发生类似事件，故障MTTR（平均修复时间）从4小时缩短至15分钟。

**关键经验**：

1. **防御性编码**：所有服务必须设置资源`limits`，并在CI/CD流水线中强制检查。
2.  **监控全覆盖**：从节点到Pod层级的资源监控需实现100%覆盖。
3.  **自动化优先**：依赖Cluster Autoscaler、HPA等自动化工具，减少人工干预。
4. **定期演练**：通过混沌工程暴露系统脆弱点，持续优化架构韧性。

通过系统化的故障处理与架构优化，Kubernetes集群的稳定性达到99.99% SLA，支撑了后续多次大促活动。
