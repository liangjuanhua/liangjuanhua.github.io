---
title: 排查 Pod 状态异常
date: 2025-04-16 17:00:36
tags: 故障指南
categories: 故障指南
---
- [Terminating](#terminating)
- [Pending](#pending)
- [ContainerCreating-Waiting](#containercreating-waiting)
- [CrashLoopBackOff](#crashloopbackoff)
- [ImagePullBackOff](#imagepullbackoff)

# Terminating

有时候删除 Pod 一直卡在 Terminating 状态，一直删不掉，可以从以下方面进行排查。

**分析思路**
一、首先我们先了解下pod的删除流程：

- APIServer 收到删除 Pod 的请求，Pod 被标记删除，处于 Terminating 状态。
- 节点上的 kubelet watch 到了 Pod 被删除，开始销毁 Pod。
- Kubelet 调用运行时接口，清理相关容器。
- 所有容器销毁成功，通知 APIServer。
- APIServer 感知到 Pod 成功销毁，检查 metadata 是否还有 finalizers，如果有就等待其它控制器清理完，如果没有就直接从 etcd 中删除 Pod 记录。

可以看出来，删除 Pod 流程涉及到的组件包含: APIServer, etcd, kubelet 与容器运行时 (如 docker、containerd)。
既然都能看到 Pod 卡在 Terminating 状态，说明 APIServer 能正常响应，也能正常从 etcd 中获取数据，一般不会有什么问题，有问题的地方主要就是节点上的操作。

二、排查思路
检查pod节点是否异常

```bash
# 查找 Terminating 的 Pod 及其所在 Node
$ kubectl get pod -o wide | grep Terminating
grafana-5d7ff8cb89-8gdtz                         1/1     Terminating   1          97d    10.10.7.150   172.20.32.15   <none>           <none>

# 检查 Node 是否异常
$ kubectl get node 172.20.32.15
NAME           STATUS      ROLES    AGE    VERSION
172.20.32.15   NotReady    <none>   182d   v1.20.6

# 查看 Node 相关事件
$ kubectl describe node 172.20.32.15
```

**检查内核日志**

```bash
dmesg
# journalctl -k
```

**存在i文件属性**
如果容器的镜像本身或者容器启动后写入的文件存在 "i" 文件属性，此文件就无法被修改删除，而删除 Pod 时会清理容器目录，但里面包含有不可删除的文件，就一直删不了，Pod 状态也将一直保持 Terminating，kubelet 报错

```bash
Sep 27 14:37:21 VM_0_7_centos kubelet[14109]: E0927 14:37:21.922965   14109 remote_runtime.go:250] RemoveContainer "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257" from runtime service failed: rpc error: code = Unknown desc = failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": Error response from daemon: container 19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257: driver "overlay2" failed to remove root filesystem: remove /data/docker/overlay2/b1aea29c590aa9abda79f7cf3976422073fb3652757f0391db88534027546868/diff/usr/bin/bash: operation not permitted
Sep 27 14:37:21 VM_0_7_centos kubelet[14109]: E0927 14:37:21.923027   14109 kuberuntime_gc.go:126] Failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": rpc error: code = Unknown desc = failed to remove container "19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257": Error response from daemon: container 19d837c77a3c294052a99ff9347c520bc8acb7b8b9a9dc9fab281fc09df38257: driver "overlay2" failed to remove root filesystem: remove /data/docker/overlay2/b1aea29c590aa9abda79f7cf3976422073fb3652757f0391db88534027546868/diff/usr/bin/bash: operation not permitted
```


**docker 17 的 bug**
docker hang 住，没有任何响应，看 event:

```bash
Warning FailedSync 3m (x408 over 1h) kubelet, 10.179.80.31 error determining status: rpc error: code = DeadlineExceeded desc = context deadline exceeded
```

**mount 的目录被其它进程占用**
dockerd 报错 device or resource busy:

```bash
May 09 09:55:12 VM_0_21_centos dockerd[6540]: time="2020-05-09T09:55:12.774467604+08:00" level=error msg="Handler for DELETE /v1.38/containers/b62c3796ea2ed5a0bd0eeed0e8f041d12e430a99469dd2ced6f94df911e35905 returned error: container b62c3796ea2ed5a0bd0eeed0e8f041d12e430a99469dd2ced6f94df911e35905: driver \"overlay2\" failed to remove root filesystem: remove /data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/merged: device or resource busy"

查找还有谁在"霸占"此目录:
$ grep 8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59 /proc/*/mountinfo
/proc/27187/mountinfo:4500 4415 0:898 / /var/lib/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/merged rw,relatime - overlay overlay rw,lowerdir=/data/docker/overlay2/l/DNQH6VPJHFFANI36UDKS262BZK:/data/docker/overlay2/l/OAYZKUKWNH7GPT4K5MFI6B7OE5:/data/docker/overlay2/l/ANQD5O27DRMTZJG7CBHWUA65YT:/data/docker/overlay2/l/G4HYAKVIRVUXB6YOXRTBYUDVB3:/data/docker/overlay2/l/IRGHNAKBHJUOKGLQBFBQTYFCFU:/data/docker/overlay2/l/6QG67JLGKMFXGVB5VCBG2VYWPI:/data/docker/overlay2/l/O3X5VFRX2AO4USEP2ZOVNLL4ZK:/data/docker/overlay2/l/H5Q5QE6DMWWI75ALCIHARBA5CD:/data/docker/overlay2/l/LFISJNWBKSRTYBVBPU6PH3YAAZ:/data/docker/overlay2/l/JSF6H5MHJEC4VVAYOF5PYIMIBQ:/data/docker/overlay2/l/7D2F45I5MF2EHDOARROYPXCWHZ:/data/docker/overlay2/l/OUJDAGNIZXVBKBWNYCAUI5YSGG:/data/docker/overlay2/l/KZLUO6P3DBNHNUH2SNKPTFZOL7:/data/docker/overlay2/l/O2BPSFNCVXTE4ZIWGYSRPKAGU4,upperdir=/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/diff,workdir=/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/work
/proc/27187/mountinfo:4688 4562 0:898 / /var/lib/docker/overlay2/81c322896bb06149c16786dc33c83108c871bb368691f741a1e3a9bfc0a56ab2/merged/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/merged rw,relatime - overlay overlay rw,lowerdir=/data/docker/overlay2/l/DNQH6VPJHFFANI36UDKS262BZK:/data/docker/overlay2/l/OAYZKUKWNH7GPT4K5MFI6B7OE5:/data/docker/overlay2/l/ANQD5O27DRMTZJG7CBHWUA65YT:/data/docker/overlay2/l/G4HYAKVIRVUXB6YOXRTBYUDVB3:/data/docker/overlay2/l/IRGHNAKBHJUOKGLQBFBQTYFCFU:/data/docker/overlay2/l/6QG67JLGKMFXGVB5VCBG2VYWPI:/data/docker/overlay2/l/O3X5VFRX2AO4USEP2ZOVNLL4ZK:/data/docker/overlay2/l/H5Q5QE6DMWWI75ALCIHARBA5CD:/data/docker/overlay2/l/LFISJNWBKSRTYBVBPU6PH3YAAZ:/data/docker/overlay2/l/JSF6H5MHJEC4VVAYOF5PYIMIBQ:/data/docker/overlay2/l/7D2F45I5MF2EHDOARROYPXCWHZ:/data/docker/overlay2/l/OUJDAGNIZXVBKBWNYCAUI5YSGG:/data/docker/overlay2/l/KZLUO6P3DBNHNUH2SNKPTFZOL7:/data/docker/overlay2/l/O2BPSFNCVXTE4ZIWGYSRPKAGU4,upperdir=/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/diff,workdir=/data/docker/overlay2/8bde3ec18c5a6915f40dd8adc3b2f296c1e40cc1b2885db4aee0a627ff89ef59/work

找到进程号后查看此进程更多详细信息:
ps -f 27187
```

# Pending

Pod 一直 Pending 一般是调度失败，调度失败的原因一般包含以下内容:
**一、节点没有足够资源分配pod**

```bash
 kubectl describe node <node-name>
```

**二、不满足节点亲和**
如果 Pod 包含 affinity（亲和性）的配置，调度器根据调度算法也可能算出没有满足条件的 Node，从而无法调度。affinity 有以下几类:
nodeAffinity: 节点亲和性，可以看成是增强版的 nodeSelector，用于限制 Pod 只允许被调度到某一部分 Node。
podAffinity:   Pod亲和性，用于将一些有关联的 Pod 调度到同一个地方，同一个地方可以是指同一个节点或同一个可用区的节点等。
podAntiAffinity: Pod 反亲和性，用于避免将某一类 Pod 调度到同一个地方避免单点故障，比如将集群 DNS 服务的 Pod 副本都调度到不同节点，避免一个节点挂了造成整个集群 DNS 解析失败，使得业务中断

**三、污点**
节点如果被打上了污点，Pod 必须要容忍污点才能调度上去:

```bash
0/5 nodes are available: 3 node(s) had taints that the pod didn't tolerate, 2 Insufficient memory.
```

通过 describe node 可以看下 Node 有哪些 Taints:

```bash
$ kubectl describe nodes host1
...
Taints:             special=true:NoSchedule
...
```

如果希望 Pod 可以调度上去，通常解决方法有两个:
1.删除污点

```bash
kubectl taint nodes host1 special-
```

2.给pod加上污点容忍

```bash
tolerations:
- key: "special"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```

**四、kube-scheduler没有正常运行**
检查 maser 上的 kube-scheduler 是否运行正常，异常的话可以尝试重启临时恢复。


# ContainerCreating-Waiting

**一、镜像问题**

**二、configmap/secret挂载问题**

**三、limit 设置太小或者单位不对**

**四、存在同名容器**

# CrashLoopBackOff

Pod 如果处于 CrashLoopBackOff 状态说明之前是启动了，只是又异常退出了。

**一、容器进程主动退出**
如果是容器进程主动退出，退出状态码一般在 0-128 之间，除了可能是业务程序 BUG，还有其它许多可能原因。内部程序崩溃或者依赖缺失等。

**二、系统 OOM**
如果是系统oom溢出，容器状态码是137，表示被 SIGKILL 信号杀死，同时内核会报错: Out of memory: Kill process。大概率是节点上部署了其它非 K8S 管理的进程消耗了比较多的内存

**三、cgroup OOM**
如果是 cgrou OOM 杀掉的进程，从 Pod 事件的下 Reason 可以看到是 OOMKilled，说明容器实际占用的内存超过 limit 了，同时内核日志会报: Memory cgroup out of memory。 可以根据需求调整下 limit。

**四、探针问题，健康检查失败**
Liveness 和 Readiness 探针配置不正确，导致容器不断重启。

# ImagePullBackOff

**一、私有镜像仓库认证失败**
仓库需要认证，配置的 Secret 不存在或者有误都会认证失败，可以先在调度的Node节点中执行docker pull命令，验证镜像是否可以拉取

**二、镜像文件损坏**
如果 push 的镜像文件损坏了，下载下来也用不了，需要重新 push 镜像文件