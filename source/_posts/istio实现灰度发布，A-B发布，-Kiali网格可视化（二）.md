---
title: istio实现灰度发布，A/B发布， Kiali网格可视化（二）
date: 2025-04-18 11:50:19
tags: istio
categories: istio
---
代码发布是软件开发生命周期中的一个重要环节，确保新功能和修复能够顺利上线。以下是几种常见的代码发布流程。在学习灰度发布之前。我们首先回忆下代码发布常用的几种方法。

**A/B（蓝绿）发布**：

蓝绿部署是一种通过维护两套独立的环境（蓝环境和绿环境）来实现零停机时间发版的方法。当前版本（蓝）和新版本（绿）是完全独立的，在绿色环境通过验证后，流量被切换到绿色环境。

**优点**：对用户无感，是最安全的发布方式，业务稳定

**缺点**：需要两套系统，对资源要求比较高，成本特别高

<img src="https://gitee.com/ljh00928/csdn/raw/master/img/2f93f6514d494bf0bcbe58e03eec3ad5.png" alt="img" style="zoom:33%;" />

**滚动式发布**：

按批次停止老版本实例，启动新版本实例。滚动更新是指逐步替换系统中的旧版本实例，而不是一次性替换所有实例。这样可以保证在更新过程中，始终有一部分实例在服务用户。

**优点**：

- 平滑过渡，减少停机时间。
- 节约资源，用户无感

**缺点**：

- 在某些情况下，可能会有部分实例同时运行不同版本，导致潜在的不一致问题。部署和回滚的速度慢

<img src="https://gitee.com/ljh00928/csdn/raw/master/img/37ecd290bded4418b4bafdd48cdc3e7e.png" alt="img" style="zoom:33%;" />

**灰度（金丝雀）发布**：

根据比例将老版本升级，例如80%用户访问是老版本，20%用户访问是新版本。金丝雀发布是将新版本应用于生产环境中的少数实例，逐步将流量引入新版本。随着监控的稳定，逐渐扩大新版本的流量范围，直到所有流量都转向新版本。

**优点**：

- 能在真实环境中测试新版本，减少风险。
- 可以通过监控和日志快速发现问题，并且迅速回滚。

**缺点**：

- 发布周期较长，逐步切换流量可能影响性能。

<img src="https://gitee.com/ljh00928/csdn/raw/master/img/280b3e0a56c94e0f9da2290db88a6ff2.png" alt="img" style="zoom:33%;" />



在现代微服务架构中，应用的更新和发布是一个高频且关键的操作。如何在不影响用户体验的前提下，安全、平稳地将新版本应用推送到生产环境，是每个开发者和运维团队必须面对的挑战。灰度发布（Gray Release）作为一种渐进式发布策略，能够有效降低发布风险。

在 Kubernetes 中，灰度发布可以通过多种方式实现，例如：

- **Deployment + Service**：修改副本数，手动控制流量切换。
- **Ingress 注解**：通过 Nginx Ingress Controller 的注解功能实现流量分割。
- **Istio**：通过服务网格实现高级流量管理。

本文重点讲解在k8s中 基于Istio的灰度发布如何实现。

**上传镜像到harbor仓库**

```
[root@harbor.cherry.com docker-client]# docker login harbor.cherry.com
[root@worker232 ~]# docker push harbor.cherry.com/istio/busybox:1.36.1
```

# 灰度发布

**流量管理之路由（权重路由模拟灰度发布）**

使用Deployment控制器部署两个版本

```bash
[root@master231 istio]# cat 01-deploy-apps.yaml 
apiVersion: v1
kind: Namespace
metadata:
  name: cherry

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: apps-v1
  namespace: cherry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: xiuxian01
      version: v1
      auther: cherry
  template:
    metadata:
      labels:
        app: xiuxian01
        version: v1
        auther: cherry
    spec:
      containers:
      - name: c1
        ports:
        - containerPort: 80
        image: harbor.cherry.com/istio/busybox:1.36.1
        command: ["/bin/sh","-c","echo 'c1' > /var/www/index.html;httpd -f -p 80 -h /var/www"]
---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: apps-v2
  namespace: cherry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: xiuxian02
      version: v2
      auther: cherry
  template:
    metadata:
      labels:
        app: xiuxian02
        version: v2
        auther: cherry
    spec:
      containers:
      - name: c2
        ports:
        - containerPort: 80
        image: harbor.cherry.com/istio/busybox:1.36.1
        command: ["/bin/sh","-c","echo 'c2' > /var/www/index.html;httpd -f -p 80 -h /var/www"]

```

手动注入Istio-proxy

```bash
查看注入之前的标签
[root@master231 istio]# kubectl -n cherry get pod --show-labels 
NAME                       READY   STATUS    RESTARTS   AGE     LABELS
apps-v1-67884c795b-256jd   1/1     Running   0          3m47s   app=xiuxian01,auther=cherry,pod-template-hash=67884c795b,version=v1
apps-v2-754f9b6f8-hx9s4    1/1     Running   0          3m47s   app=xiuxian02,auther=cherry,pod-template-hash=754f9b6f8,version=v2

将Istio sidecar代理注入注入指定名称空间下
[root@master231 istio]# istioctl kube-inject -f 01-deploy-apps.yaml | kubectl -n cherry  apply -f -
namespace/cherry unchanged
deployment.apps/apps-v1 configured
deployment.apps/apps-v2 configured

再次查看标签.注入istio之后。很明显pod数量也变成了两个。
[root@master231 istio]# kubectl -n cherry get pod --show-labels
NAME                       READY   STATUS    RESTARTS   AGE   LABELS
apps-v1-77f9877969-2c54w   2/2     Running   0          38s   app=xiuxian01,auther=cherry,pod-template-hash=77f9877969,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=xiuxian01,service.istio.io/canonical-revision=v1,version=v1
apps-v2-64bcbb594b-jbdqd   2/2     Running   0          38s   app=xiuxian02,auther=cherry,pod-template-hash=64bcbb594b,security.istio.io/tlsMode=istio,service.istio.io/canonical-name=xiuxian02,service.istio.io/canonical-revision=v2,version=v2
```

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/c292cd30926d4b13a6ad781d92a35685.png)


通过查看标签，部署的v1和v2他们version不同。但是auther是相同的。所以我们编写svc的时候。可以分别通过version关联不同版本的pod。使用auther同时关联两个pod

```bash
[root@master231 istio]# cat 02-svc-apps.yaml 
apiVersion: v1
kind: Service
metadata:
  name: apps-svc-v1
  namespace: cherry
spec:
  selector:
    #关联v1版本
    version: v1
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    name: http

---

apiVersion: v1
kind: Service
metadata:
  name: apps-svc-v2
  namespace: cherry
spec:
  selector:
    #关联v2版本
    version: v2
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    name: http

---

apiVersion: v1
kind: Service
metadata:
  name: apps-svc-all
  namespace: cherry
spec:
  selector:
    #关联两个版本
    auther: cherry
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    name: http
```

查看svc详情详细,不难发现通过auther关联的pod。ep列表有两个ip

```bash
[root@master231 istio]# kubectl -n cherry describe svc
Name:              apps-svc-all
Namespace:         cherry
Labels:            <none>
Annotations:       <none>
Selector:          auther=cherry
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.200.95.30
IPs:               10.200.95.30
Port:              http  80/TCP
TargetPort:        80/TCP
Endpoints:         10.100.1.18:80,10.100.2.17:80
Session Affinity:  None
Events:            <none>


Name:              apps-svc-v1
Namespace:         cherry
Labels:            <none>
Annotations:       <none>
Selector:          version=v1
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.200.99.39
IPs:               10.200.99.39
Port:              http  80/TCP
TargetPort:        80/TCP
Endpoints:         10.100.1.18:80
Session Affinity:  None
Events:            <none>


Name:              apps-svc-v2
Namespace:         cherry
Labels:            <none>
Annotations:       <none>
Selector:          version=v2
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.200.229.240
IPs:               10.200.229.240
Port:              http  80/TCP
TargetPort:        80/TCP
Endpoints:         10.100.2.17:80
Session Affinity:  None
Events:            <none>
```

部署VirtualService。将来访问apps-svc-all-vs的时候。会代理apps-svc-v1和apps-svc-v2

```bash
[root@master231 istio]# cat 03-vs-apps-svc-all.yaml 
apiVersion: networking.istio.io/v1beta1
# apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: apps-svc-all-vs
  namespace: cherry
spec:
  # 指定vs关联的后端svc名称
  hosts:
  - apps-svc-all
  # 配置http配置
  http:
    # 定义路由信息
  - route:
      # 定义目标
    - destination:
        host: apps-svc-v1
      # 指定权重
      weight: 90
    - destination:
        host: apps-svc-v2
      weight: 10
```

部署客户端测试

```bash
[root@master231 istio]# cat 04-deploy-client.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: apps-client
  namespace: cherry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client-test
  template:
    metadata:
      labels:
        app: client-test
    spec:
      containers:
      - name: c1
        image: registry.cn-hangzhou.aliyuncs.com/yinzhengjie-k8s/apps:v1 
        command:
        - tail
        - -f
        - /etc/hosts
```

客户端注入Istio-proxy

```bash
[root@master231 istio]# istioctl kube-inject -f 04-deploy-client.yaml | kubectl -n cherry apply -f -
deployment.apps/apps-client configured
```

测试

```bash
[root@master231 istio]# kubectl -n cherry get pod
NAME                          READY   STATUS    RESTARTS   AGE
apps-client-98fddd875-hp259   2/2     Running   0          3m24s
apps-v1-77f9877969-2c54w      2/2     Running   0          104m
apps-v2-64bcbb594b-jbdqd      2/2     Running   0          104m


[root@master231 istio]# kubectl exec -it apps-client-98fddd875-hp259 -n cherry -- sh
/ # while true; do curl http://apps-svc-all;sleep 0.1;done
c1
c1
c1
c1
c1
c2
c1
c1
c1
c1
c1
c2
c1
c1
c1
c1
c1
c1
c1
c1
c1
c1
c1
/ # while true; do curl http://apps-svc-all; sleep 0.1; done > 1.txt
/ # cat 1.txt | sort | uniq -c
     72 c1
     10 c2
```

此时很明显。通过VirtualService资源定义svc权重比。已经达到了灰度发布的一个效果

# A/B测试

流量管理之基于用户匹配（定向路由模拟A/B测试）

通过请求头信息。将版本精确打入到指定的区域。这里我们将header信息name是cherry的。全部打入v1版本。否则全部打入v2

```bash
[root@master231 istio]# cat 05-vs-apps-svc-all.yaml 
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: apps-svc-all-vs
  namespace: cherry
spec:
  hosts:
  - apps-svc-all
  http:
    # 定义匹配规则
  - match:
      # 基于header信息匹配将其进行路由，header信息自定义即可。
    - headers:
        # 匹配用户名包含"cherry"的用户，这个KEY是咱们自定义的。
        username:
          # "eaxct"关键词是包含，也可以使用"prefix"进行前缀匹配。
          exact: cherry
    route:
    - destination:
        host: apps-svc-v1
  - route:
    - destination:
        host: apps-svc-v2
```

测试

```bash
[root@master231 istio]# kubectl exec -it apps-client-98fddd875-hp259 -n cherry -- sh
/ #  while true; do curl -H  "username:cherry" http://apps-svc-all;sleep 0.1;done  # 添加用户认证的header信息
c1
c1
c1
c1
c1
c1
c1
^C
/ #  while true; do curl  http://apps-svc-all;sleep 0.1;done # 不添加用户认证
c2
c2
c2
c2
c2
c2
c2
```

# 网格可视化

 Kiali 是一个为 Istio 提供图形化界面和丰富观测功能的 Dashboard 的开源项目，其名称源于希腊语，意思是望远镜。用户利用 Kiali 可以监测网格内服务的实时工作状态，管理Istio的网络配置，快速识别网络问题。但是从Istio 1.7开始，默认不安装控制面板Kiali等组件，所以需要用户自行单独安装控制面板Kiali及相关的组件。

```bash
[root@master231 istio-1.17.3]#  kubectl apply -f samples/addons

[root@master231 istio-1.17.3]# kubectl apply -f samples/addons/extras
```

配置Kiali控制面板对外访问

查看kiali服务，发现其类型为ClusterIP，没有对外暴露端口，无法从外部访问：

```bash
[root@chon istio-1.9.0]# kubectl get service kiali -n istio-system
```

将类型改NodePort

```bash
[root@master231 istio-1.17.3]# kubectl -n istio-system edit svc kiali
...
spec:
  clusterIP: 10.200.5.137
  clusterIPs:
  - 10.200.5.137
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - appProtocol: http
    name: http
    nodePort: 30003      #添加对外暴露端口
    port: 20001
    protocol: TCP
    targetPort: 20001
  - appProtocol: http
    name: http-metrics
    nodePort: 31367
    port: 9090
    protocol: TCP
    targetPort: 9090
  selector:
    app.kubernetes.io/instance: kiali
    app.kubernetes.io/name: kiali
  sessionAffinity: None
  type: NodePort    #修改网络类型
status:
  loadBalancer: {}

```

访问

```bash
http://10.0.0.233:30003/
```

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/18403ac5291248dfb0e0760c0d36d0df.png)

