---
title: Spring Cloud + Nacos + K8S 零影响发布方案Spring Cloud + Nacos + K8S 零影响发布方案
date: 2025-04-18 11:31:47
tags: k8s
categories: 
  - 云原生
---
#  问题描述

在生产环境中使用 springcloud 框架，由于服务更新过程中，容器服务会被直接停止，部分请求仍被分发到终止的容器，导致服务出现500错误，这部分错误请求数据占用比较少，因为Pod滚动更新都是一对一。因为部分用户会产生服务器错误的情况，考虑使用优雅的终止方式，将错误请求降到最低，直至滚动更新不影响用户。这里结合nacos使用来分析。

在 K8s 的滚动升级中，比如 5 个 Pod 服务在升级过程中，会先启动一半左右（比如：3 个新的启动），然后下线一部分服务……直到所有的旧服务被新服务完全替代，简单粗暴的理解滚动升级。如果我们不涉及 Nacos 还好，因为 K8s 会保证在升级过程中，因为负载的情况很有可能在升级过程中会一部分请求打到旧服务里，但是如果在旧服务准备关闭服务时，旧情求还没返回回去的话就会出现 HTTP 连接关闭情况等一些不可预测的意外发生，导致本次请求的业务失败，这是在生产上绝不能出现的事故。针对 K8s 的优雅关闭问题，我们可以继续往下看，下面会介绍 Nacos & K8s 一个结合优雅关闭的方案。

我们来再谈谈 Nacos 在这里如果无优雅关闭的话会出现的情况，其实和 K8s 的本质很类似，假如我们已经解决了 K8s 的优雅关闭问题，那和 Nacos 之间又有什么联系呢？

我们可以想象下，还是举例上面的 5 个 Pod 的情景，在一个 Pod 启动时，服务的也自然会注册到 Nacos 上去，同理可得，在服务关闭时，Nacos 注册列表里服务也自然会被下线。那么类似的情况也会出现，如果说此时的情求打到旧服务上面，但是由于 Nacos 有监听时间（默认 30s）拉取一次最新情况，以及在每个 Pod 服务里本地也有一份缓存映射表（也有一个窗口时间更新），所以很有可能在这个窗口期之内，还有一些的旧的请求访问负载到旧服务里，但是这里会出现两种情况

K8s Pod 服务已下线，但是 Nacos 在窗口时间之内注册列表未更新，导致请求达到一个根本不存在的旧服务里 旧请求已经打到旧服务里，但是高峰期时，程序处理较慢，还没来及返回响应体，服务就被关闭了 以上这两种情况都会导致本次请求出现失败，生产上更是无语~ 所以我们针对 Nacos 的优雅关闭情况也会有一个解决方案，见“Nacos & K8s 优雅关闭方案”

# 一、思路

为了解决这个问题, 我们将利用 springboot 的 graceful shutdown 功能和 nacos 的主动下线功能来解决这个问题. 具体思路如下:

1、让新的服务先启动起来，注册到注册中心，等待客户端发现新服务。

2、把待下线的服务从注册中心下线，等待客户端刷新服务列表。

3、把待下线的服务优雅停机。

# 二、实现方式

## **1.创建从 nacos 中下线副本的API**

我们通过创建自定义名为 `deregister` 的 `endpoint` 来通知 `nacos` 下线副本

```java
import com.alibaba.cloud.nacos.NacosDiscoveryProperties;
import com.alibaba.cloud.nacos.registry.NacosRegistration;
import com.alibaba.cloud.nacos.registry.NacosServiceRegistry;
import lombok.extern.log4j.Log4j2;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.actuate.endpoint.annotation.Endpoint;
import org.springframework.boot.actuate.endpoint.annotation.ReadOperation;
import org.springframework.stereotype.Component;

@Component
@Endpoint(id = "deregister")
@Log4j2
public class LemesNacosServiceDeregisterEndpoint {

    @Autowired
    private NacosDiscoveryProperties nacosDiscoveryProperties;

    @Autowired
    private NacosRegistration nacosRegistration;

    @Autowired
    private NacosServiceRegistry nacosServiceRegistry;

    /**
     * 从 nacos 中主动下线，用于 k8s 滚动更新时，提前下线分流流量
     *
     * @param
     * @return com.lenovo.lemes.framework.core.util.ResultData<java.lang.String>
     * @author Yujie Yang
     * @date 4/6/22 2:57 PM
     */
    @ReadOperation
    public String endpoint() {
        String serviceName = nacosDiscoveryProperties.getService();
        String groupName = nacosDiscoveryProperties.getGroup();
        String clusterName = nacosDiscoveryProperties.getClusterName();
        String ip = nacosDiscoveryProperties.getIp();
        int port = nacosDiscoveryProperties.getPort();

        log.info("deregister from nacos, serviceName:{}, groupName:{}, clusterName:{}, ip:{}, port:{}", serviceName, groupName, clusterName, ip, port);

        // 设置服务下线
        nacosServiceRegistry.setStatus(nacosRegistration, "DOWN");

        return "success";
    }

}
```

## 2.支持 Graceful Shutdown

Spring Boot 原生支持已经支持优雅停机。

我们需要添加下面的配置，超时需要根据平时请求的耗时来定，可以稍微大一点也没关系。我们只需要在 `bootstrap.yaml` 中添加一下配置即可。

```java
server:
  # 开启优雅下线
  shutdown: graceful

spring:
  lifecycle:
    # 优雅下线超时时间
    timeout-per-shutdown-phase: 5m
# 暴露 shutdown 接口
management:
  endpoint:
    shutdown:
      enabled: true
  endpoints:
    web:
      exposure:
        include: shutdown
```

## 3.配置K8s 

有了上面两个 API, 接下来就配置到 k8s 上

1.K8S容器本身有一个就绪探针配置，当就绪探针返回正常，则开始删除旧POD。

2.terminationGracePeriodSeconds 如果关闭应用的时间超过 10 分钟, 则向容器发送 kill 信号, 防止应用长时间下线不了

terminationGracePeriodSeconds要大于preStop+spring.lifecycle.timeout-per-shutdown-phase，可以设置大一点没什么关系。

3.preStop 先执行下线操作, 等待30s, 留够通知到其他应用的时间, 然后执行 graceful shutdown 关闭应用。

```bash
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lemes-service-common
  labels:
    app: lemes-service-common
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lemes-service-common
#  strategy:
#    type: RollingUpdate
#    rollingUpdate:
##     replicas - maxUnavailable < running num  < replicas + maxSurge
#      maxUnavailable: 1
#      maxSurge: 1
  template:
    metadata:
      labels:
        app: lemes-service-common
    spec:
#      容器重启策略 Never Always OnFailure
#      restartPolicy: Never
#     在pod优雅终止时，定义延迟发送kill信号的时间，此时间可用于pod处理完未处理的请求等状况。默认单位是秒
#     如果关闭时间超过10分钟， 则向容器发送kill信号
      terminationGracePeriodSeconds: 600
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - podAffinityTerm:
                topologyKey: "kubernetes.io/hostname"
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - lemes-service-common
              weight: 100
#          requiredDuringSchedulingIgnoredDuringExecution:
#            - labelSelector:
#                matchExpressions:
#                  - key: app
#                    operator: In
#                    values:
#                      - lemes-service-common
#              topologyKey: "kubernetes.io/hostname"
      volumes:
        - name: lemes-host-path
          hostPath:
            path: /data/logs
            type: DirectoryOrCreate
        - name: sidecar
          emptyDir: { }
      containers:
        - name: lemes-service-common
          image: 10.176.66.20:5000/lemes-cloud/lemes-service-common-server:v0.1
          imagePullPolicy: Always
          volumeMounts:
            - name: lemes-host-path
              mountPath: /data/logs
            - name: sidecar
              mountPath: /sidecar
          ports:
            - containerPort: 80
          resources:
#           资源通常情况下的占用
            requests:
              memory: '2048Mi'
#           资源占用上限
            limits:
              memory: '4096Mi'
          livenessProbe:
            httpGet:
#             指定 Kubernetes 访问的健康检查 URL 路径
              path: /actuator/health/liveness
              port: 80
            initialDelaySeconds: 5
#           探针可以连续失败的次数
            failureThreshold: 10
#           探针超时时间
            timeoutSeconds: 10
#           多久执行一次探针查询
            periodSeconds: 10
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 80
            failureThreshold: 30
            timeoutSeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 80
            initialDelaySeconds: 5
            timeoutSeconds: 10
            periodSeconds: 10
          lifecycle:
            preStop:
              exec:
#应用关闭操作：1. 从 nacos 下线，2. 等待30s, 保证 nacos 通知到其他应用 2.触发 springboot 的 graceful shutdown
                command:
                  - sh
                  - -c
                  - curl http://127.0.0.1/actuator/deregister;sleep 30;curl -X POST http://127.0.0.1/actuator/shutdown;
```
