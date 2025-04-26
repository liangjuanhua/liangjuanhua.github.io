---
title: Kubernetes 常用的网络插件
date: 2025-04-16 16:39:41
tags: 网络篇  
categories: 网络篇
---
 上篇内容跟大家简单聊了k8s网络模型原理。分别围绕着容器、Pod、Service、网络策略等展开了详细的讲解。这次想跟大家聊聊k8s的CNI网络插件。

CNI 是 Kubernetes 网络模型的核心组件，它是一个插件接口，允许用户选择和配置网络插件来管理 Pod 的网络。CNI 插件提供了网络连接、IP 地址分配、路由控制等基本功能。

**常见的 CNI 插件包括：**

Flannel：用于实现简单的网络隧道。

Calico：支持网络策略、跨节点网络路由等功能。

Weave：简化的网络配置，支持跨节点通信。

Cilium：基于 eBPF 技术的高性能网络插件，支持深度安全控制。

接触过K8S的同学，大致都听说过Flannel和Calico两种网络模型。这里就我们主要讲解Flannel和Calico的工作模式和原理。

## 一、Flannel

Flannel 是一个简单的网络插件，设计目的是为 Kubernetes 提供一个易于部署和配置的网络解决方案。它的目标是简化网络设置，适合那些对网络复杂度要求不高的 Kubernetes 集群。Flannel 基本上是一个 Overlay 网络 解决方案，它在每个节点上创建一个虚拟网络，并通过隧道技术（如 VXLAN、UDP、Host-GW）来实现跨节点的 Pod 网络通信。

- UDP
- VXLAN
- Host-gw

### 1.1 flannel-udp

UPD模式Flannel最早实现的一种方式，也是性能最差的，目前已被弃用。但是这种方式也是最直接，最容易理解的方式，所以我们从这种方式开始介绍。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/9eab445bb6cb4f5490ef462a71c6c993.png)

上图是Flannel-UDP模式的原理图。flannel0设备是一个TUN设备，它的作用非常简单，就是在系统内核和用户应用程序之间传包；flanneld进程的职责，就是封装和解封装。数据包是如何从Node1中的container-1容器发送到Node2的container-2容器的呢？

1.数据包从container-1，来到了网桥docker0上，由于数据包的目的地址不属于网桥的网段，所以数据包经由docker0网桥，出现在宿主机上。

2.在宿主机的路由表中，去往100.96.0.0/16网段的包经由flannel0处理。flannel0收到数据包之后，将数据包送到flanneld进程，flanneld进程会对数据包封装成一个UDP数据包，src和dst地址分别为两个容器对应的宿主机的地址。这样，数据包就可以到达Node2了。

3.数据包到达Node2的8285端口，即Node2上的flanneld进程，会被执行解封装操作，之后数据包被发送到TUN设备，即flannel0设备。剩下的事情就简单了，数据包经过docker0网桥到达container-2。

### 1.2 Flannel-vxlan

经过上面的介绍，大家对Flannel-UDP模式大致了解了吧，那聪明的你们已经猜到为什么Flannel-UDP被弃用了吧？没错，因为效率太低了，数据包每次经过flannel0设备，都会经过内核态-用户态-内核态的这一顿折腾。

Flannel-VXLAN方案用VXLAN技术替代了flannel0设备，让数据包能够在内核态上实现数据包的封装和解封装。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/bcc35c4be7a94477ad793a524d2ca1bf.png)

Flannel-VXLAN网络模型如图所示，你会发现，这和Flannel-UDP基本上的是一样。事实也的确如此，Flannel-VXLAN是Flannel-UDP的升级版。这里需要交代一下他们之间的不同点。

1.Flannel-UDP的TUN设备flannel0，升级成了VXLAN的VTEP设备。数据包的封装和解封装在内核态就能完成。

2.数据包的格式中，增加了VXLAN Header，这个Header的作用和Flannel-UDP的数据包中的dport:8285的作用是一样的，当数据包来到Node2时，操作系统能根据VXLAN Header，把数据包直接给到flannel.1设备。

### 1.3 Flannel-host-gw

此时，你肯定会说，Flannel-VXLAN虽然效率提高了，但是还是用到了隧道技术，效率还是会受到影响，能不能不用隧道技术呢？答案是能。接下来我们继续探索Flannel-host-gw网络模型，一个基于三层的网络方案。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/92996b23553e4231af1533a9c9c10779.png)

Flannel-host-gw网络模型，相比较之前的两个网络模型，隧道设备确实没有了，取而代之的是一堆路由规则。那，数据包又是怎么从container1到container2的呢？

1.当数据包从container1到了网桥之后，通过Host网络栈的路由表，发现去container2的路已经指明，经由eth0，达到Node2（10.168.0.3/24）即可。

2.当数据包到了Node2之后，通过Host网络栈的路由表，找到cni0网桥，container2自然也就找到了。

肉眼可见，Flannel-host-gw的性能确实提高了很多，那为什么还要用Flannel-VXLAN呢？原因很明显，Flannel-host-gw只支持宿主机在二层连通的网络，并且，K8S的规模不能太大，否则每台机器的路由表就太多了。

## 二、Calico

Calico 是一个功能强大的网络插件，提供了高效的 路由 网络架构，并支持 网络策略（Network Policy），适合大规模、复杂的 Kubernetes 集群。它不仅适用于 Overlay 网络，还支持 BGP（边界网关协议）路由，提供高性能的网络连接。

### 2.1 Calico（非IPIP模式）

实际上Calico网络模型的解决方案，几乎和Flannel-host-gw是一样的。不同的是Flannel-host-gw使用etcd来维护主机的路由表，而Calico则使用BGP（边界网关协议）来维护主机的路由表。BGP协议的定义看着有点高深，换成通俗的说法，大家可以理解为在每个边界网关都会都运行着一个小程序，它们会交换各自的路由信息，将需要的信息更新到自己的路由表里。BGP这个能力正好可以取代Flannel-host-gw利用Etcd维护主机上路由表的功能，并且更为强大。

除了BGP之外，Calico另外一个不同之处就在于它不需要维护一个网桥。其中BGP Client和Felix的作用是和K8S集群其他节点交换路由信息，并更新Host网络栈的路由信息。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/a32ece678fe048a087b6c1b593979938.png)

由于没有了网桥设备，每个对设备Host网络栈的这一端，需要配置一条路由规则，将目的地址为对应Container的数据包转入该对设备。对应的路由如下所示：

```
10.233.1.2 dev cali9c02e56 scope link
```


数据包是如何从Container1走到Container3的呢？过程基本上和Flannel-host-gw无异了。唯一区别就是数据包进出容器，不再依赖网桥，而是直接通过宿主机路由表找到容器的另一端对设备。

### 2.2 Calico（IPIP模式）

Calico听着挺强大的，实则和Flannel-host-gw一样，只支持宿主机二层联通的情况。假设Container1和Container3的宿主机在不同的子网，那通过二层网络是无法将数据包传到下一跳的地址的。如图7所示，Calico会在Node1创建这样一条路由规则：

```
10.233.2.0/16 via 192.168.2.2 eth0
```

此时问题就出现了，下一跳是192.168.2.2，和Node1不在一个子网里，根本就找不到。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/600e6f5cb9a9459ca77357ffb02e154f.png)

Calico的IPIP模式解决了上述问题，在每一台宿主机上，都会增加一个tunl0设备（IP隧道设备），并且会对应增加如下一条路由策略。

```
10.233.2.0/16 via 192.168.2.2 tunl0
```

这样一来，Container1去往Container3的数据包就会经过tunl0设备的处理，tunl0设备会在源IP报头之外新增一个外部IP报头，拿本例来说，这个外部IP报头的src和dst分别为Node1和Node2的IP，这样，数据包就伪装成了从Node1发到Node2的数据包。当数据包到达Node2之后，Node2上的tunl0会把外部IP报头拿掉，从而拿到原始的IP包。

我知道，聪明的你此时肯定会有一个更好的想法，为什么不在Router1和Router2上也用BGP协议的方式，同步容器的IP路由信息呢？这样宿主机上不就可以不用tunl0设备了么。这个方法确实很好，并且在一些场景也得到了应用。

## 三、Flannel vs Calico 区别

| **特性**           | **Flannel**              | **Calico**                       |
| ------------------ | ------------------------ | -------------------------------- |
| **网络架构**       | Overlay 网络（隧道模式） | 基于路由（BGP 或 IP-in-IP）      |
| **性能**           | 性能较低（因为使用隧道） | 高性能，接近原生网络性能         |
| **网络策略支持**   | 不支持网络策略           | 强大的网络策略支持（细粒度控制） |
| **配置和管理**     | 配置简单，适合快速部署   | 配置较复杂，但灵活性更高         |
| **适用集群规模**   | 适合中小规模集群         | 适合大规模集群或跨数据中心部署   |
| **安全控制**       | 无网络策略控制           | 提供丰富的安全控制和流量隔离     |
| **支持的网络模式** | VXLAN, UDP, Host-GW      | BGP, IP-in-IP, VXLAN 等多种模式  |

##  四、总结

- **Flannel** 是一个轻量级的 Overlay 网络插件，适合中小型 Kubernetes 集群，特别是在对网络性能和安全要求不高的情况下。它安装和配置简单，但不支持网络策略，功能相对基础。
- **Calico** 提供更强大的功能，特别适合需要高性能、大规模、复杂安全控制和跨数据中心连接的 Kubernetes 集群。它不仅提供高效的网络路由（BGP），还支持细粒度的网络安全控制（通过网络策略

**参考文献**

[1] 本文的图片均引自张磊老师的《深入剖析Kubernetes》

[2] Linux Bridge（网桥基础） [https://quemingfei.com/archives/linuxbridge-wang-qiao-ji-chu-](https://link.zhihu.com/?target=https%3A//quemingfei.com/archives/linuxbridge-wang-qiao-ji-chu-)

[3] 维基百科 [https://en.wikipedia.org/wiki](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki)