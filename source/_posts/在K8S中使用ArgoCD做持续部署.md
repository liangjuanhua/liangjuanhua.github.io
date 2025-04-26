---
title: 在K8S中使用ArgoCD做持续部署
date: 2025-04-18 11:09:08
tags: CICD
categories: CICD
---
## 一、了解argocd

ArgoCD是一个基于Kubernetes的GitOps持续交付工具，应用的部署和更新都可以在Git仓库上同步实现，并自带一个可视化界面。本文介绍如何使用Git+Argocd方式来实现在k8s中部署和更新应用服务。关于ci这一块这里不多介绍。主要讲解argocd如何实现cd持续部署。在开始前，需要部署一套k8s集群，可参考本文连接：https://blog.csdn.net/m0_69326428/article/details/144375315?spm=1011.2415.3001.5331

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/b819ac46c7674fa5b52da3668eac0397.png)

### 工作原理

ArgoCD 的核心理念是 GitOps，即以 Git 仓库作为单一的真理源，通过自动化的方式将仓库中的应用配置同步到 Kubernetes 集群中。

1. **定义应用**: 用户在 Git 仓库中定义应用的 Kubernetes 资源清单，并将这些清单文件提交到 Git 仓库。
2. **创建 ArgoCD Application**: 在 ArgoCD 中创建一个 `Application` 资源，该资源描述了应用在 Git 仓库中的位置，以及在 Kubernetes 集群中部署的位置。
3. **同步状态监控**: ArgoCD Controller 持续监控 Git 仓库中的配置，并与当前集群状态进行对比。每次检测到 Git 仓库中的应用配置发生变化时，Controller 会自动更新集群中的资源，保持与 Git 仓库的一致性。
4. **自动同步与手动同步**: ArgoCD 支持自动同步和手动同步。自动同步模式下，一旦检测到 Git 仓库有变化，ArgoCD 会自动更新 Kubernetes 集群中的资源。而在手动同步模式下，用户需要手动触发同步操作。
5. **回滚功能**: 如果应用更新导致问题，ArgoCD 提供了回滚功能，用户可以轻松恢复到先前的状态

CD 流水线有两种模式：Push 和 Pull

### Push 模式

目前大多数 CI/CD 工具都使用基于 Push 的部署模式，例如 Jenkins。这种模式一般都会在 CI 流水线运行完成后执行一个命令（比如 kubectl）将应用部署到目标环境中。

这种 CD 模式的缺陷很明显：

需要在环境安装配置额外管理工具（比如 kubectl）；
需要 Kubernetes 对其进行授权；
需要云平台授权；
无法感知部署状态。也就无法感知期望状态与实际状态的偏差，需要借助额外的方案来保障一致性。
Kubernetes 集群或者云平台对 CI 系统的授权凭证在集群或云平台的信任域之外，不受集群或云平台的安全策略保护，因此 CI 系统很容易被当成非法攻击的载体。

### Pull 模式

Pull 模式会在目标环境中安装一个 Agent，例如在 Kubernetes 集群中就靠 Operator 来充当这个 Agent。Operator 会周期性地监控目标环境的实际状态，并与 Git 仓库中的期望状态进行比较，如果实际状态不符合期望状态，Operator 就会更新基础设施的实际状态以匹配期望状态。

只有 Git 的变更可以作为期望状态的唯一来源，除此之外，任何人都不可以对集群进行任何更改，即使你修改了，也会被 Operator 还原为期望状态，这也就是传说中的不可变基础设施。

目前基于 Pull 模式的 CD 工具有 Argo CD，Flux CD 以及 ks-devops。



## 二、部署argocd

github地址：https://github.com/argoproj/argo-cd

**准备环境**

```bash
#下载argocd client
wget https://github.com/argoproj/argo-cd/releases/download/v2.12.7/argocd-linux-amd64

#权限
chmod u+x argocd-linux-amd64

#移动可执行目录
mv ./argocd-linux-amd64 /usr/local/bin/argocd

#验证 
argo version

#准备yaml文件
wget https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**修改svc类型**

为了方便测试。将svc类型改成NodePort。实际工作中建议使用ingress

```bash
[root@master231 ~]# vim install.yaml
...
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd-server
    app.kubernetes.io/part-of: argocd
  name: argocd-server
spec:
  # 增加 type: NodePort
  type: NodePort
  ports:
  - name: http
    port: 80
    # 该位置增加访问端口 300xxx (30000-32000)任意 我们设置成30080
    nodePort: 30080
    protocol: TCP
    targetPort: 8080
  - name: https
    port: 443
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/name: argocd-server
...
```

**部署**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f install.yaml
```

**查看pod状态**

```bash
kubectl get all -n argocd
```

**访问**

```bash
https://10.0.0.231:30080/login
```

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/a659614bebb341bdbaf3996b043b77a0.png)


**查看密码**

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
密码：****  
账号：admin
```

**argocd客户端命令行工具修改密码**

```bash
[root@master231 bin]# argocd login 10.0.0.231:30080
WARNING: server certificate had error: tls: failed to verify certificate: x509: cannot validate certificate for 10.0.0.231 because it doesn't contain any IP SANs. Proceed insecurely (y/n)? y
Username: admin
Password: 
'admin:login' logged in successfully
Context '10.0.0.231:30080' updated

[root@master231 bin]# argocd account update-password
*** Enter password of currently logged in user (admin): 
*** Enter new password for user admin: 
*** Confirm new password for user admin: 
Password updated
Context '10.0.0.231:30080' updated

[root@lc-master-1 ~]# argocd logout  192.168.0.71:8082
Logged out from '192.168.0.71:8082'
[root@master231 bin]# argocd login 10.0.0.231:30080
WARNING: server certificate had error: tls: failed to verify certificate: x509: cannot validate certificate for 10.0.0.231 because it doesn't contain any IP SANs. Proceed insecurely (y/n)? y
Username: admin
Password: 
'admin:login' logged in successfully
Context '10.0.0.231:30080' updated

```

## 三、web界面介绍

### **设置介绍**

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/fb09126f952b41dfa43415c39f4e38f5.png)


### **添加仓库地址**

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/21dfe8d196044229ac7c1e1b7ef9a211.png)


![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/17d2ef035c4047f8870a530c6ae199fe.png)


添加成功

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/4b752038d5c24eba9feb7de5d3e08a98.png)


## 四、创建应用

### 通过 CLI 来创建应用

在仓库https://gitee.com/ljh00928/test_cherry里有个app目录，里面有个 [myapp-deployment.yaml](https://gitee.com/zouzou_busy/devops_test/blob/master/app/myapp-deployment.yaml?spm=a2c6h.13046898.publish-article.13.25876ffa0ywKH4&file=myapp-deployment.yaml) 文件 和 [myapp-service.yaml](https://gitee.com/zouzou_busy/devops_test/blob/master/app/myapp-service.yaml?spm=a2c6h.13046898.publish-article.14.25876ffa0ywKH4&file=myapp-service.yaml) 文件，用来演示我们 argo cd 的功能

myapp-deployment.yaml 

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        test: c1
    spec:
      containers:
      - image: yankay/dao-2048:latest
        name: myapp
        ports:
        - containerPort: 8008
```

myapp-service.yaml

```bash
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  ports:
  - port: 8009
    targetPort: 8009
  type: NodePort
  selector:
    app: myapp
```

创建应用

```bash
#查看帮助手册
argocd app create --help

#部署应用
argocd app create app01 --repo https://gitee.com/ljh00928/test_cherry.git --path app --dest-server https://kubernetes.default.svc --dest-namespace demo1
```

### 通过 UI 创建应用

同步策略：

自动同步允许 Argo CD 自动将 Git 仓库中的应用程序状态同步到 Kubernetes 集群中。

手动同步要求用户通过 Argo CD UI 或 CLI 手动触发同步操作。

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/fa2f42bc21b64d08bfb9473796e45113.png)


由于 Argo CD 支持部署应用到多集群，所以如果你要将应用部署到外部集群的时候，需要先将外部集群的认证信息注册到 Argo CD 中，如果是在内部部署（运行 Argo CD 的同一个集群，默认不需要配置），直接使用 `https://kubernetes.default.svc` 作为应用的 K8S APIServer 地址即可。

首先列出当前 `kubeconfig` 中的所有集群上下文：

```bash
[root@master231 ~]# kubectl config get-contexts 
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin    orbstack
```

从列表中选择一个上下文名称并将其提供给 `argocd cluster add CONTEXTNAME`，比如对于 `orbstack` 上下文，运行：

```bash
[root@master231 ~]# argocd cluster list
SERVER                          NAME        VERSION  STATUS      MESSAGE  PROJECT
https://kubernetes.default.svc  in-cluster  1.23     Successf]ul

argocd cluster add orbstack
```

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/3e7e3fcd71bd4a4fab48065455fd8230.png)


查看yaml文件

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/537b8072b9ee45b58fe05f53d88c8cfc.png)


```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app02
spec:
  destination:
    name: ''
    namespace: demo2
    server: 'https://kubernetes.default.svc'
  source:
    path: app
    repoURL: 'https://gitee.com/ljh00928/test_cherry.git'
    targetRevision: HEAD
  project: default
  syncPolicy:
    automated: null
    syncOptions:
      - CreateNamespace=true
```

填写完以上信息后，点击页面左上方的 Create 安装，即可创建 app02 应用，创建完成后可以看到当前应用的处于 `OutOfSync` 状态：

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/d4a267c8db484a7589b621c7271df127.png)


==Argo CD 默认情况下每 3 分钟会检测 Git 仓库一次，用于判断应用实际状态是否和 Git 中声明的期望状态一致，如果不一致，状态就转换为 OutOfSync。默认情况下并不会触发更新，除非通过 syncPolicy 配置了自动同步==

**SYNC OPTIONS（同步策略）**

```bash
spec:
  syncPolicy:
    syncOptions:
    - Validate=false
    - CreateNamespace=true
    - PruneLast=true
    - ApplyOutOfSyncOnly=true
    - Replace=false
    - SkipDryRunOnMissingResource=true
```

常见的同步选项包括：

- `Validate=false`: 禁用资源的服务器端验证。这在某些自定义资源（CRD）可能尚未完全定义时非常有用。
- `CreateNamespace=true`: 如果命名空间不存在，自动创建它。
- `PruneLast=true`: 在同步过程中最后执行 `prune` 操作，以确保所有资源已经创建。
- `ApplyOutOfSyncOnly=true`: 仅应用那些状态不同步的资源。
- `Replace=false`: 使用 `kubectl apply` 而不是 `kubectl replace` 来更新资源。
- `SkipDryRunOnMissingResource=true`: 在资源缺失时跳过 `dry-run` 检查

## 五、部署应用

上面我们创建好了应用，但还没有部署，所以 namespace、pod、deployment、svc 都没有

### 使用 CLI 同步

应用创建完成后，我们可以通过如下所示命令查看其状态

```bash
[root@master231 ~]# argocd app get app01
Name:               argocd/app01   #应用名称
Project:            default
Server:             https://kubernetes.default.svc  #部署的服务
Namespace:          demo1  #部署的ns
URL:                https://10.0.0.231:30080/applications/app01 
Source:
- Repo:             https://gitee.com/ljh00928/test_cherry.git  #资源仓库
  Target:           
  Path:             app   #仓库里的资源路径
SyncWindow:         Sync Allowed
Sync Policy:        Manual
Sync Status:        OutOfSync from  (30c6f26)  #仓库里的资源路径
Health Status:      Missing  #健康状态

GROUP  KIND        NAMESPACE  NAME       STATUS     HEALTH   HOOK  MESSAGE
       Service     demo1      myapp-svc  OutOfSync  Missing        
apps   Deployment  demo1      myapp      OutOfSync  Missin
```

因为 app01 是我们通过命名行创建的，ns 写的是 demo1，没有设置自动创建。如果你集群上没有这个命名空间，需要先手动创建

```bash
[root@master231 ~]# kubectl create ns demo1
namespace/demo1 created
```

应用程序状态为初始 `OutOfSync` 状态，因为应用程序尚未部署，并且尚未创建任何 Kubernetes 资源。要同步（部署）应用程序，可以执行如下所示命令

```bash
#同步应用app01
[root@master231 ~]# argocd app sync app01
TIMESTAMP                  GROUP        KIND   NAMESPACE                  NAME    STATUS    HEALTH        HOOK  MESSAGE
2025-03-25T15:12:33+08:00            Service       demo1             myapp-svc  OutOfSync  Missing              
2025-03-25T15:12:33+08:00   apps  Deployment       demo1                 myapp  OutOfSync  Missing              
2025-03-25T15:12:33+08:00            Service       demo1             myapp-svc    Synced  Healthy              
2025-03-25T15:12:33+08:00            Service       demo1             myapp-svc    Synced   Healthy              service/myapp-svc created
2025-03-25T15:12:33+08:00   apps  Deployment       demo1                 myapp  OutOfSync  Missing              deployment.apps/myapp created
2025-03-25T15:12:33+08:00   apps  Deployment       demo1                 myapp    Synced  Progressing              deployment.apps/myapp created

Name:               argocd/app01
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          demo1
URL:                https://10.0.0.231:30080/applications/app01
Source:
- Repo:             https://gitee.com/ljh00928/test_cherry.git
  Target:           
  Path:             app
SyncWindow:         Sync Allowed
Sync Policy:        Manual
Sync Status:        Synced to  (30c6f26)
Health Status:      Progressing

Operation:          Sync
Sync Revision:      30c6f26bc59ce7f0605caac7c43e5316c55c89ce
Phase:              Succeeded
Start:              2025-03-25 15:12:33 +0800 CST
Finished:           2025-03-25 15:12:33 +0800 CST
Duration:           0s
Message:            successfully synced (all tasks run)

GROUP  KIND        NAMESPACE  NAME       STATUS  HEALTH       HOOK  MESSAGE
       Service     demo1      myapp-svc  Synced  Healthy            service/myapp-svc created
apps   Deployment  demo1      myapp      Synced  Progressing        deployment.apps/myapp created
```

此命令从 Git 仓库中检索资源清单并执行 `kubectl apply` 部署应用，执行上面命令后 guestbook 应用便会运行在集群中了，现在我们就可以查看其资源组件、日志、事件和评估其健康状态了。

```bash
#再次查看app01状态
[root@master231 ~]# argocd app get app01
Name:               argocd/app01
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          demo1
URL:                https://10.0.0.231:30080/applications/app01
Source:
- Repo:             https://gitee.com/ljh00928/test_cherry.git
  Target:           
  Path:             app
SyncWindow:         Sync Allowed
Sync Policy:        Manual
Sync Status:        Synced to  (30c6f26)
Health Status:      Progressing   #状态为 Progressing（进行中）了

GROUP  KIND        NAMESPACE  NAME       STATUS  HEALTH       HOOK  MESSAGE
       Service     demo1      myapp-svc  Synced  Healthy            service/myapp-svc created
apps   Deployment  demo1      myapp      Synced  Progressing        deployment.apps/myapp created
```

等一会在去查看状态

```bash
[root@master231 ~]# argocd app get app01
Name:               argocd/app01
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          demo1
URL:                https://10.0.0.231:30080/applications/app01
Source:
- Repo:             https://gitee.com/ljh00928/test_cherry.git
  Target:           
  Path:             app
SyncWindow:         Sync Allowed
Sync Policy:        Manual
Sync Status:        Synced to  (8a1ee3f)
Health Status:      Healthy    #状态为 Healthy（健康）的了

GROUP  KIND        NAMESPACE  NAME       STATUS  HEALTH   HOOK  MESSAGE
       Service     demo1      myapp-svc  Synced  Healthy        service/myapp-svc unchanged
apps   Deployment  demo1      myapp      Synced  Healthy        deployment.apps/myapp unchanged
```

 然后查看 pod、deploy、svc

```bash
[root@master231 ~]# kubectl -n demo1 get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/myapp-6449b755f5-5fkzf   1/1     Running   0          2m26s

NAME                TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/myapp-svc   NodePort   10.200.5.218   <none>        8009:31922/TCP   22m

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/myapp   1/1     1            1           22m

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/myapp-5f748b96c    0         0         0       22m
replicaset.apps/myapp-6449b755f5   1         1         1       7m18s
```

### 使用 UI 界面同步

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/0857ff9730414e6691e334755b9af0b3.png)


查看资源状态

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/fc3ce3c2686443b297732806089192d7.png)


也可以查看日志、event 等信息

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/332cdfbf3fd5492e951a22bbfc673e86.png)


查看 pod、deploy、svc。都运行正常

```bash
[root@master231 ~]# kubectl -n demo2 get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/myapp-6449b755f5-4tl7m   1/1     Running   0          7m20s

NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/myapp-svc   NodePort   10.200.164.64   <none>        8009:31185/TCP   7m20s

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/myapp   1/1     1            1           7m20s

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/myapp-6449b755f5   1         1         1       7m20s
```

## 六、更新应用

上面我们已经部署好了两个应用 app01 和 app02，现在来更改一下 myapp-deployment.yaml 文件，将镜像改为green

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/0278af6638ad4285b655e435b59bb389.png)


再次点击sync同步按钮，可以看见有两个rs,一个副本数为 0，一个副本数为 1

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/7fd8c69b053c470a8f0d26f6023913fc.png)


```bash
[root@master231 ~]# kubectl -n demo2 get rs
NAME               DESIRED   CURRENT   READY   AGE
myapp-6449b755f5   0         0         0       19m
myapp-65c5d9cf87   1         1         1       4m3s
```

## 七、回滚

上面我们的 app02 已经有两个版本了，现在最新的是 geen版本，我们也可以可以回滚到第一个版本

现在是这个版本

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/ea8d0b2632244d7ba6a20dde06e58dad.png)


在回滚的时候需要禁用 AUTO-SYNC 自动同步，点击历史和回滚。找到要回滚的版本，点击 Rollback

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/b52f3c52db6040829d9a054f990bf1dc.png)


![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/f27b292e1868437dab42657ce0cef295.png)


这时候已经回滚到第一个版本了

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/d86d9c05e6cf4201a764845d0f109750.png)


```bash
#回滚前
[root@master231 ~]# kubectl -n demo2 get rs
NAME               DESIRED   CURRENT   READY   AGE
myapp-6449b755f5   0         0         0       32m
myapp-65c5d9cf87   1         1         1       16m

#回滚后
[root@master231 ~]# kubectl -n demo2 get rs
NAME               DESIRED   CURRENT   READY   AGE
myapp-6449b755f5   1         1         1       35m
myapp-65c5d9cf87   0         0         0       19m
```

查看 pod，svc，deployment

```bash
[root@master231 ~]# kubectl -n demo2 get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/myapp-6449b755f5-252l9   1/1     Running   0          3m10s

NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/myapp-svc   NodePort   10.200.164.64   <none>        8009:31185/TCP   36m

NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/myapp   1/1     1            1           36m

NAME                               DESIRED   CURRENT   READY   AGE
replicaset.apps/myapp-6449b755f5   1         1         1       36m
replicaset.apps/myapp-65c5d9cf87   0         0         0       20m
```