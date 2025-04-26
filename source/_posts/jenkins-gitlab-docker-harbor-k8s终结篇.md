---
title: jenkins+gitlab+docker+harbor+k8s终结篇
date: 2025-04-18 11:02:12
tags: CICD
categories: CICD
---
 之前我们已经把相关环境，持续集成这一块都实现了。详细内容可查看我cicd专栏前三篇的内容。

[kubeadm 部署k8s-CSDN博客](https://blog.csdn.net/m0_69326428/article/details/144375315)

 [harbor部署-CSDN博客](https://blog.csdn.net/m0_69326428/article/details/144376600)

[Jenkins+gitlab持续集成_gitlab jenkins 集成-CSDN博客](https://blog.csdn.net/m0_69326428/article/details/144470530)

本篇内容主要是讲解持续集成和持续交付是如何实现和部署的。

**概念**

持续交付建立在持续集成的基础上，通过自动化的流程确保软件可以随时随地进行部署。

**流程**
 这时，持续交付后的代码已经在主分支上了，这处于某个版本的待发布的状态，随时可以将开发环境的功能部署到生产环境中（部署到生成环境前还需要在测试环境性能测试、回归测试、自动化测试、人工测试等），运行脚本构建打包应用，通过自动化部署工具部署到生产环境运行应用，监控生产环境指标，如出现问题和错误，可以触发手动或自动回滚，如系统正常，则定期回顾，收集反馈，优化，并持续改进

**流程图**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/33e75aa499d6441e9e997bd210cff040.png)

**机器规划如下**

| 设备名称   | ip地址     | 配置                    |
| ---------- | ---------- | ----------------------- |
| master231  | 10.0.0.231 | 2核，4GiB，系统盘 20GiB |
| worken232  | 10.0.0.232 | 2核，4GiB，系统盘 20GiB |
| worken233  | 10.0.0.233 | 2核，4GiB，系统盘 20GiB |
| jenkins211 | 10.0.0.211 | 2核，4GiB，系统盘 20GiB |
| gitlab212  | 10.0.0.212 | 4核，4GiB，系统盘 20GiB |
| harbor     | 10.0.0.250 | 2核，4GiB，系统盘 20GiB |

## 一、编写dockerfile

```
[root@gitlab212 project]# pwd
/root/project

[root@gitlab212 project]# vim Dockerfile 
FROM registry.cn-hangzhou.aliyuncs.com/yinzhengjie-k8s/apps:v1

EXPOSE 80

WORKDIR /usr/share/nginx/html/

ADD . /usr/share/nginx/html/
```

本地打镜像测试

```
[root@gitlab212 project]# docker image build -t yiliao:v1 .
```

运行镜像

```
[root@gitlab212 project]# docker run -d -p 88:80  yiliao:v1
```

访问

> http://10.0.0.212:88/

![img](https://gitee.com/ljh00928/csdn/raw/master/img/55dcdcf431a14bbd9bd889ff73a342ff.png)

## 二、集成harbor

1.启动harbor仓库

```
[root@harbor.oldboyedu.com harbor]# docker-compose down -t 0
[root@harbor.oldboyedu.com harbor]# docker-compose up -d
```

2.harbor新建仓库

![img](https://gitee.com/ljh00928/csdn/raw/master/img/675b129dd52e4148a2b55c4137344aee.png)

 3.jenkins编写shell

```
docker login -u admin -p 1 harbor.cherry.com
docker image build -t harbor.cherry.com/yiliao/yiliao:v1 .
docker push harbor.cherry.com/yiliao/yiliao:v1
```

![img](https://gitee.com/ljh00928/csdn/raw/master/img/551110df74e84b5396f6dbe650f56e0c.png)

 4.推送故障排查

![img](https://gitee.com/ljh00928/csdn/raw/master/img/6a30c2a630dd403fb81d770862faf5fa.png)

没有docker命令，只需要在jenkins上安装docker 



jenkins上做harbor的hosts解析

> [root@jenkins211 ~]# echo 10.0.0.250 harbor.cherry.com >>/etc/hosts



 x509证书问题

![img](https://gitee.com/ljh00928/csdn/raw/master/img/553aa352eb7549d5bcb57541d7ef717f.png)

```
创建harbor客户端证书存放目录
[root@jenkins211 ~]# mkdir -pv /etc/docker/certs.d/harbor.cherry.com

harbor上需要准备客户端证书
[root@harbor certs]# ll
total 20
drwxr-xr-x 5 root root 4096 Dec 10 08:26 ./
drwxr-xr-x 4 root root 4096 Dec 10 08:45 ../
drwxr-xr-x 2 root root 4096 Dec 10 08:31 ca/
drwxr-xr-x 2 root root 4096 Dec 18 05:55 docker-client/
drwxr-xr-x 2 root root 4096 Dec 10 08:32 harbor-server/

[root@harbor.oldboyedu.com certs]# openssl x509 -inform PEM -in harbor-server/harbor.cherry.com.crt -out docker-client/harbor.cherry.com.cert
#查看证书一致性
[root@harbor.oldboyedu.com certs]# md5sum docker-client/harbor.cherry.com.cert harbor-server/harbor.cherry.com.crt
cp harbor-server/harbor.cherry.key /docker-client/
cp ca/ca.crt docker-client/

[root@harbor certs]# cd docker-client/
[root@harbor docker-client]# ll
total 20
drwxr-xr-x 2 root root 4096 Dec 18 05:55 ./
drwxr-xr-x 5 root root 4096 Dec 10 08:26 ../
-rw-r--r-- 1 root root 2041 Dec 18 05:55 ca.crt
-rw-r--r-- 1 root root 2134 Dec 18 05:38 harbor.cherry.com.cert
-rw------- 1 root root 3272 Dec 18 05:54 harbor.cherry.com.key

#拷贝证书到jenkins
[root@jenkins211 ~]# scp harbor.cherry.com:/softwares/harbor/certs/docker-client/* /etc/docker/certs.d/harbor.cherry.com
[root@jenkins211 ~]# ll /etc/docker/certs.d/harbor.cherry.com
```

登录harbor仓库

```
[root@jenkins211 harbor.cherry.com]# docker login -u admin -p 1 harbor.cherry.com
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

 5.harbor仓库查看镜像

这时候镜像已经推送到harbor仓库了

![img](https://gitee.com/ljh00928/csdn/raw/master/img/f796bdb8b1474bdd97dc40f3b178bc60.png)

## 三、jenkins一键更新k8s项目

1.新建yiliao名称空间

```
[root@master231 01-yiliao]# cat 01-ns-yiliao.yaml 
apiVersion: v1
kind: Namespace
metadata:
  labels:
    school: cherry
    class: cherry
  name: yiliao
```

2.使用deploy控制器编写资源清单

```
[root@master231 01-yiliao]# cat 02-deploy-yiliao.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-yiliao
  namespace: yiliao
spec:
  replicas: 3
  selector:
    matchLabels:
      apps: yiliao
  template:
    metadata:
      labels:
        apps: yiliao
    spec:
       containers:
       - name: yiliao
         image: harbor.cherry.com/yiliao/yiliao:v1
```

3.编写svc

```
[root@master231 01-yiliao]# cat 03-svc-yiliao.yaml 
apiVersion: v1
kind: Service
metadata:
  name: yiliao-svc
  namespace: yiliao
spec:
  type: NodePort
  selector:
    apps: yiliao
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

4.部署

```
#创建资源
[root@master231 01-yiliao]# kubectl apply -f .

#指定名称空间查看
[root@master231 01-yiliao]# kubectl get svc,po -n yiliao 
```

5.创建secret

```
kubectl create secret docker-registry harbor-secret \
  --docker-server=harbor.cherry.com \
  --docker-username=admin \
  --docker-password=1 \
  --namespace=yiliao
```

创建工作负载

```
[root@master231 yiliao]# ll
total 20
drwxr-xr-x  2 root root 4096 Dec 18 15:30 ./
drwx------ 10 root root 4096 Dec 18 15:30 ../
-rw-r--r--  1 root root  103 Dec 18 14:44 01-ns-yiliao.yaml
-rw-r--r--  1 root root  378 Dec 18 15:20 02-deploy-yiliao.yaml
-rw-r--r--  1 root root  191 Dec 18 15:30 03-svc-yiliao.yaml


[root@master231 yiliao]# kubectl apply -f .
namespace/yiliao created
deployment.apps/deploy-yiliao created
service/yiliao-svc created


//所有节点都需要harbor客户端证书，不然镜像拉取会失败
```

5.访问

> http://10.0.0.232:30080/

![img](https://gitee.com/ljh00928/csdn/raw/master/img/68d16e255d0c43fea5ef537a6c352659.png)

##  四、更新项目

1.jenkins安装kubectl工具

```
我本地已经下载了，这里就直接上传到jenkins中

#将kubectl放到PATH环境变量
[root@jenkins211 ~]# mv kubectl-1.23.17 /usr/local/sbin/kubectl
[root@jenkins211 ~]# chmod +x /usr/local/sbin/kubectl
```

2.pod创建的时候会携带https证书，所以我们需要拷贝231节点证书到jenkins

```
[root@jenkins211 ~]# mkdir -pv ~/.kube/
[root@jenkins211 ~]# scp 10.0.0.231:/root/.kube/config ~/.kube/
```

3.gitlab修改项目

![img](https://gitee.com/ljh00928/csdn/raw/master/img/4fcf891879064c54b24f90c429963041.png)

4.jenkins构建项目，测试

```
docker login -u admin -p 1 harbor.cherry.com
docker image build -t harbor.cherry.com/yiliao/yiliao:v3 .
docker push harbor.cherry.com/yiliao/yiliao:v3
kubectl -n yiliao set image deploy deploy-yiliao yiliao=harbor.cherry.com/yiliao/yiliao:v3
```

![img](https://gitee.com/ljh00928/csdn/raw/master/img/b853e828b3de476ebaa60355be6fbbbc.png)

 查看harbor仓库，我们打的镜像都成功推上来了

![img](https://gitee.com/ljh00928/csdn/raw/master/img/7fbd2f4bfc4e4fafb2ce1730ba04eac2.png)

 访问

![img](https://gitee.com/ljh00928/csdn/raw/master/img/035923010dab43f89a143957b3f4ade4.png)

## 五、自动构建

我们每次都需要手动更改jenkins版本号，这是件麻烦的事，我们可以选择git参数构建的方式，这样就不需要手动更改版本号

1.git参数填写

![img](https://gitee.com/ljh00928/csdn/raw/master/img/24c425ebcfe0448da204dd89c10d5d52.png)

 2.修改jenkins中shell脚本

```
docker login -u admin -p 1 harbor.cherry.com
docker image build -t harbor.cherry.com/yiliao/yiliao:$release .
docker push harbor.cherry.com/yiliao/yiliao:$release
kubectl -n yiliao set image deploy deploy-yiliao yiliao=harbor.cherry.com/yiliao/yiliao:$release
```

4.jenkins选择版本构建

5.harbor仓库查看最新镜像