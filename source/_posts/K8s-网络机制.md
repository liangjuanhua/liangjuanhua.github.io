---
title: K8s 网络机制
date: 2025-04-16 16:43:58
tags: 网络篇  
categories: 网络篇
---
## 概述

Service 作为 K8s 中的一等公民，其承载了核心容器网络的访问管理能力，包括：

- 暴露/访问一组 Pod 的能力
- Pod 访问集群内、集群外服务
- 集群外客户端访问集群内 Pod 的服务

无论是作为应用的开发者还是使用者，一般都需要先经过 Service 才会访问到真正的目标 Pod。因此熟悉 Service 网络管理机制将会使我们更加深入理解 K8s 的容器编排原理，

本文将从 K8s 中容器网络、Service/Pod 关联、Service 类型、kube-proxy 模式、Ingress 等方面，说明 Service 的网络机制。



# 一、Service类型

| 类型         | 应用场景                                                     |
| ------------ | ------------------------------------------------------------ |
| CLusterIP    | K8S集群内部相互访问                                          |
| NodePort     | K8S集群外部实现访问                                          |
| LoadBalancer | 云环境中使用，比如K8S在阿里云，腾讯云，京东云，华为云等。对应的云产品都有自己的负载均衡器产品 |
| ExternerName | SVC代理的服务并不在K8S集群内部，而是在K8S集群外部            |

 ![img](https://gitee.com/ljh00928/csdn/raw/master/img/7dd7d9de57bd46bda0f4f2fba82e6ad5.png) **ClusterIP**

ClusterIP 表示在 K8s 集群内部通过 service.spec.clusterIP 进行访问，之后经过 kube-proxy 负载均衡到目标 Pod。

无头服务 (Headless Service)：
 当指定 Service 的 ClusterIP = None 时，则创建的 Service 不会生成 ClusterIP，这样 Service 域名在解析的时候，将直接解析到对应的后端 Pod (一个或多个)，某些业务如果不想走 Service 默认的负载均衡，则可采用此种方式 直连 Pod。

> // service.spec.publishNotReadyAddresses：表示是否将没有 ready 的 Pods 关联到 Service，默认为 false。设置此字段的主要场景是为 StatefulSet 的 Service 提供支持，使之能够为其 Pod 传播 SRV DNS 记录，以实现对等发现。

 当没有指定 service.type 时，默认类型为 ClusterIP。

```
apiVersion: v1kind: Servicemetadata:  name: headless-servicespec:  selector:    app: nginx  ports:    - protocol: TCP      port: 80      targetPort: 80  type: ClusterIP # 默认类型，可省略  clusterIP: None # 指定 ClusterIP = None  publishNotReadyAddresses: true # 是否关联未 ready pods
```

## NodePort

当业务需要从 K8s 集群外访问内部服务时，通过 NodePort 方式可以先将访问流量转发到对应的 Node IP，然后再通过 service.spec.ports[].nodePort 端口，通过 kube-proxy 负载均衡到目标 Pod。

> Service NodePort 默认端口范围：30000-32767，共 2768 个端口。
>  可通过 kube-apiserver 组件的 `--service-node-port-range` 参数进行配置。
>
> 
>
> [root@master231 ~]# vim /etc/kubernetes/manifests/kube-apiserver.yaml 
>  ...
>  spec:
>   containers:
>   \- command:
>    \- kube-apiserver
>    \- --service-node-port-range=3000-50000  # 进行添加这一行即可

```
apiVersion: v1kind: Servicemetadata:  name: nodeport-servicespec:  selector:    app: nginx  ports:    - nodePort: 30800   #worker节点的端口映射      port: 8080        #对应的svc的port端口      protocol: TCP           targetPort: 80    #对应的pod端口  type: NodePort
```

##  **LoadBalancer**

上面的 NodePort 方式访问内部服务，需要依赖具体的 Node 高可用，如果节点挂了则会影响业务访问，LoadBalancer 可以解决此问题。

具体来说，LoadBalancer 类型的 Service 创建后，由具体是云厂商或用户实现 externalIP (service.status.loadBalancer) 的分配，业务直接通过访问 externalIP，然后负载均衡到目标 Pod。

```
apiVersion: v1kind: Servicemetadata:  name: my-servicespec:  selector:    app.kubernetes.io/name: MyApp  ports:    - protocol: TCP      nodePort: 30931      port: 80      targetPort: 9376  clusterIP: 10.0.171.239  type: LoadBalancerstatus:  loadBalancer:    ingress:    - ip: 192.0.2.127
```

##  ExternalName

当业务需要从 K8s 内部访问外部服务的时候，可以通过 ExternalName 的方式实现。Demo 如下

具体来说，service.spec.externalName 字段值会被解析为 DNS 对应的 CNAME 记录，之后就可以访问到外部对应的服务了

```
apiVersion: v1kind: Service  #无selector，无endpointsmetadata:  name: my-service    namespace: prodspec:  type: ExternalName  externalName: my.database.example.com
```

#  二、容器网络机制

K8s 将一组逻辑上紧密相关的容器，统一抽象为 Pod 概念，以共享 Pod Sandbox 的基础信息，如 Namespace 命名空间、IP 分配、Volume 存储（如 hostPath/emptyDir/PVC）等，因此讨论容器的网络访问机制，实际上可以用 Pod 访问机制代替。

根据 Pod 在集群内的分布情况，可将 Pod 的访问方式主要分为两种：

- 同一个 Node 内 Pod 访问
- 跨 Node 间 Pod 访问

## 同一个 Node 内访问

同一个 Node 内访问，表示两个或多个 Pod 落在同一个 Node 宿主机上，这种 Pod 彼此间访问将不会跨 Node，通过本机网络即可完成通信。

具体来说，Node 上的运行时如 Docker/containerd 会在启动后创建默认的网桥 cbr0 (custom bridge)，以连接当前 Node 上管理的所有容器 (containers)。当 Pod 创建后，会在 Pod Sandbox 初始化基础网络时，调用 CNI bridge 插件创建 veth-pair（两张虚拟网卡），一张默认命名 eth0 (如果 hostNetwork = false，则后续调用 CNI ipam 插件分配 IP)。另一张放在 Host 的 Root Network Namespace 内，然后连接到 cbr0。当 Pod 在同一个 Node 内通信访问的时候，直接通过 cbr0 即可完成网桥转发通信。

> 访问流程：
>
> - 首先运行时如 Docker/containerd 创建 cbr0；
> - Pod Sandbox 初始化调用 CNI bridge 插件创建 veth-pair（两张虚拟网卡）；
> - 一张放在 Pod Sandbox 所在的 Network Namespace 内（CRI containerd 默认传的参数为 eth0）；
> - 另一张放在 Host 的 Root Network Namespace 内，然后连接到 cbr0；
> - Pod 在同一个 Node 内访问直接通过 cbr0 网桥转发；
>
> // docker中默认网桥名为docker 0，k8s中默认网桥名为cni 0

![img](https://gitee.com/ljh00928/csdn/raw/master/img/e61a893bdeb7451aad260b4a5e15fa91.png)

## 跨 Node 间 Pod 访问

跨 Node 间访问，Pod 访问流量通过 veth-pair 打到 cbr0，之后转发到宿主机 eth0，之后通过 Node 之间的路由表 Route Table 进行转发。到达目标 Node 后进行相同的过程，最终转发到目标 Pod 上。

> 访问流程：
>
> - 用户访问某个 Pod 时，首先访问的是 Kubernetes 服务。
> - 服务通过 kube-proxy 将请求转发到后端的 Pod。
> - 由于服务是跨节点的，kube-proxy 会根据不同的访问策略（如 iptables 或 IPVS）选择后端的 Pod。如果 Pod 位于不同的节点，流量会通过节点之间的网络进行转发。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/eb084beaad664485a2bd7de0f97c0d46.png)

## Pod 的定义

Pod 是 Kubernetes 中最小的部署单位。Pod 中包含一个或多个容器（通常是单个容器）。这些容器共享同一个网络命名空间。

> 特点：
>
> - 每个 Pod 会被 Kubernetes 分配一个 唯一的 IP 地址，这个 IP 地址仅在集群内部有效。
> - Pod 的生命周期是短暂的，一个 Pod 会随着应用的部署而创建，随着应用的销毁而被删除。
> - Pod 通常由 Deployment、StatefulSet 或 DaemonSet 等控制器进行管理，控制器负责确保
> - Pod 的数量和健康状态。

## **Service 的定义**

Service 是一个抽象，它定义了一组具有相同功能的 Pod，并为这些 Pod 提供一个固定的访问入口。Pod 的 IP 地址是动态的，可能会因为 Pod 的重启或者节点的变动而变化，Service 通过对外提供一个稳定的访问接口来解决这个问题。 

> 特点：
>
> - ClusterIP：这是最常见的类型，它为 Service 分配一个虚拟 IP 地址，Pod 通过该 IP 地址暴露给其他 Pod 或客户端。
> - NodePort：在 ClusterIP 的基础上，Service 会在每个节点上开放一个静态端口，从而使服务可以通过 NodeIP:NodePort 进行访问。
> - LoadBalancer：在 NodePort 的基础上，Service 会向云提供商请求创建一个外部负载均衡器，从而为外部客户端提供访问。
> - Headless Service：如果没有为 Service 提供 ClusterIP（即 clusterIP: None），则 Service 仅提供一组 Pod 的 DNS 名称而没有虚拟 IP。这种类型常用于 StatefulSet 等需要稳定 DNS 的场景。

## 两者之间关系

Service 通过指定选择器 (selector) 去选择与目标 Pod 匹配的标签 (labels)，找到目标 Pod 后，建立对应的 Endpoints 对象。当感知到 Service/Endpoints/Pod 对象变化时，创建或更新 Service 对应的 Endpoints，使得 Service selector 与 Pod labels 始终达到匹配的状态。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/a993fc7a1e934a389aa83b4fdc735b55.png)

# 四、Kube-proxy多种模式

 在 Kubernetes 中，kube-proxy 是一种负责服务代理和负载均衡的关键组件，它允许 Kubernetes 集群中的 Pod 通过 Service 访问其他 Pod。kube-proxy 在多个网络模式下工作，控制流量如何从客户端发送到 Service 以及 Service 后端的 Pod。

Kubernetes 提供了三种 kube-proxy 的工作模式，每种模式使用不同的方式来管理流量的路由和负载均衡。以下是这三种模式的详细解释：

- Userspace 模式
- iptables 模式
- ipvs 模式

## **Userspace 模式**

这是kube-proxy的最早版本，也是 Kubernetes 在早期版本中使用的默认模式，这里不多讲解，感兴趣小伙伴可以自行百度。

## **iptables 模式**

K8s 中当前默认的 kube-proxy 模式，核心逻辑是使用 iptables 中 PREROUTING 链 nat 表，实现 Service => Endpoints (Pod IP) 的负载均衡。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/1a22d7a413db450193c1e612fc587486.png)

> 工作原理：
>
> - 在 iptables 模式下，kube-proxy 会通过管理 iptables 规则来处理流量的路由。当客户端请求一个 Service 时，kube-proxy 会将流量重定向到后端的 Pod。每个 Service 在 iptables 中都有一条规则，客户端的请求会根据规则被路由到某个 Pod。
> - iptables 可以将请求直接转发到 Pod，而不需要经过用户空间进程，因此性能得到了显著提高。

##  **ipvs 模式**

ipvs是 Linux 内核提供的一个虚拟服务器模块，专门用于高效的负载均衡。相比iptables，ipvs提供了更强大、灵活的负载均衡能力。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/dd08f07685384e34bc52763b4510bf37.png)

> 工作原理：
>
> - 在 ipvs 模式下，kube-proxy 使用ipvs来实现负载均衡。ipvs为每个 Service 创建虚拟的 IP 地址（VIP），然后将流量根据负载均衡算法转发到后端的 Pod。
> - ipvs支持多种负载均衡算法，如轮询（Round Robin）、最少连接（Least Connection）、加权轮询（Weighted Round Robin）等。

| 模式     | 工作原理                                                     | 优点                                         | 缺点                                                       | 使用场景                                 |
| -------- | ------------------------------------------------------------ | -------------------------------------------- | ---------------------------------------------------------- | ---------------------------------------- |
| iptables | 使用 Linux 内核的 `iptables` 规则来进行流量路由和负载均衡。  | 性能优越，简化配置。                         | 随着规则数增多，性能会下降。负载均衡算法简单，通常是轮询。 | 中小规模的集群，适合大多数生产环境。     |
| ipvs     | 使用 Linux 内核的 `ipvs` 模块来进行高效的负载均衡，支持多种负载均衡算法。 | 高效负载均衡，支持更多的调度算法，性能优秀。 | 配置复杂，需要内核支持 `ipvs` 模块。                       | 大规模集群，高流量环境，复杂负载均衡需求 |

# 五、Ingress

之前学习的svc资源，如果遇到多个服务要监听80端口时很明显无论哪种类型都无法实现，如果非要实现，就得在K8S集群外部部署一个LB设备，来代理到对应svc资源。而ingress就可以很好的解决这个问题。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/5b1151206cd2452c824e6c99189bd104.png)

## **什么是ingress呢？**

> - 所谓的ingress指的是一种规则，基于用户访问的请求头路由到正确的svc。说白了就是7层代理
> - 可惜K8S只是实现了ingress定义规则，这个规则被记录到etcd中，但并没有具体实现此功能，因此需要自行安装相应的附加组件(ingress-nginx,trafik,...)

## **svc和ingress的区别？**

> - ingress和svc的区别是，svc只能实现4层的代理。而ingress实现了7层的代理。

## **Ingress Controller和ingress区别？**

> - ingress是定义域名到svc的解析规则，好比nginx.conf配置文件。
> - 而ingress-controller适用于执行规则的程序，好比nginx程序。

## **Ingress Controller和内置的Pod控制器有啥区别呢？**

> - 内置的Pod控制器，比如ds,sts,deploy,jobs,cj,rs,rc等都是用来控制Pod的副本数量。
> - 而Ingress Controller是用来解析ingress规则的，两者并没有任何关系

## **demo**

创建工作负载

```
[root@master231 ~]# cat deploy-apps.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-apps-v1
spec:
  replicas: 3
  selector:
    matchLabels:
      apps: v1
  template:
    metadata:
      labels:
        apps: v1
    spec:
      containers:
      - name: c1
        image: nginx:1.26.2
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: svc-apps
spec:
  selector:
    apps: v1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

创建ingress资源

```
[root@master231 ~]# cat 01-apps-ingress.yaml 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: apps-ingress
spec:
  rules:
  - host: apps.cherry.com
    http:
      paths:
      - backend:
          service:
            name: svc-apps
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific
```