---
title: EFK采集k8s日志
date: 2025-04-16 16:06:43
tags: 日志篇
categories: 日志篇
---
​    在 Kubernetes 集群中，需要全面了解各个 pod 应用运行状态、故障排查和性能分析。但由于 Pod 是动态创建和销毁的，其日志分散且存储不持久，因此需要通过集中式日志采集方案，将日志收集到统一的平台并配置日志可视化分析和监控告警，以实现日志的可追溯性、实时监控和高效分析，从而提升运维效率和系统可靠性。

官网地址：[日志架构 | Kubernetes](https://kubernetes.io/zh/docs/concepts/cluster-administration/logging/)

## 1.日志采集方案

**方案一：ds控制器**

每个节点有且仅有一个日志采集的Pod。并不需要注入原有的Pod。DaemonSet + nodeSelector调度到指定节点。DaemonSet 相对资源占用要小很多，但扩展性、租户隔离性受限，比较适用于功能单一或业务不是很多的集群。

优点：节省资源

缺点：需要配置RBAC权限

**方案二：sidecar**

每个pod再起一个容器，安装filebeat采集数据，使用emptyDir共享pod中日志。Sidecar 相对资源占用较多，但灵活性以及多租户隔离性较强，建议大型的 K8s 集群或作为 PaaS 平台为多个业务方服务的集群使用该方式。

优点：可以直接和业务容器共享存储和网络，将日志采集到目的端。

缺点: 如果一个节点有五十个pod，那就要注入五十个filebeat采集，造成资源浪费

**方案三：开发自实现**

让程序本身支持日志采集功能。

优点：无需运维接入，只需配置对应目标接口

缺点：跨部门沟通，费时间

![img](https://gitee.com/ljh00928/csdn/raw/master/img/6e5e2bc0104c49a5a5946abebc903b59.png)

> **总结：**
>
>   **1. DaemonSet一般在中小型集群中使用**
>
>    **2. Sidecar推荐在超大型的集群中使用**
>
>   **3. 业务直写推荐在日志量极大的场景中使用**

**三种方案优缺点：** 

![img](Untitled.assets/4e47cb38331b43bda5414eaab67214ed.png)

因为方案一在业界使用更为广泛，并且官方也更为推荐，所以我们基于方案一来做k8s的日志采集。

## 2.创建单节点elasticsearch的yaml文件

```
apiVersion: v1
kind: Namespace
metadata:
  name: efk

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
  namespace: efk
  labels:
    k8s-app: elasticsearch
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: elasticsearch
  template:
    metadata:
      labels:
        k8s-app: elasticsearch
    spec:
      containers:
      # 指定需要安装的ES版本号
      - image: docker.elastic.co/elasticsearch/elasticsearch:7.17.22
        name: elasticsearch
        resources:
          limits:
            cpu: 2
            memory: 3Gi
          requests:
            cpu: 0.5 
            memory: 500Mi
        env:
          # 配置集群部署模式，此处我由于是实验，配置的是单点
          - name: "discovery.type"
            value: "single-node"
          - name: ES_JAVA_OPTS
            value: "-Xms256m -Xmx256m" 
        ports:
        - containerPort: 9200
          name: db
          protocol: TCP
        volumeMounts:
        - name: elasticsearch-data
          mountPath: /usr/share/elasticsearch/data
      volumes:
      - name: elasticsearch-data
        persistentVolumeClaim:
          claimName: es-pvc

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: es-pvc
  namespace: efk
spec:
  storageClassName: "nfs-sc"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi

---

apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: efk
spec:
  ports:
  - port: 9200
    protocol: TCP
    targetPort: 9200
  selector:
    k8s-app: elasticsearch
```

## 3.创建kibana的yaml文件

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: efk
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: kibana
  template:
    metadata:
      labels:
        k8s-app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:7.17.22
        resources:
          limits:
            cpu: 2
            memory: 2Gi
          requests:
            cpu: 0.5 
            memory: 500Mi
        env:
          - name: ELASTICSEARCH_HOSTS
            value: http://elasticsearch.efk.svc.cherry.com:9200
          - name: I18N_LOCALE
            value: zh-CN
        ports:
        - containerPort: 5601
          name: ui
          protocol: TCP

---
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: efk
spec:
  type: NodePort
  ports:
  - port: 5601
    protocol: TCP
    targetPort: ui
    nodePort: 5601
  selector:
    k8s-app: kibana
```

## 4.以DaemonSet的方式创建filebeat的yaml文件

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
data:
  filebeat.yml: |-
    filebeat.config:
      inputs:
        path: ${path.config}/inputs.d/*.yml
        reload.enabled: false
      modules:
        path: ${path.config}/modules.d/*.yml
        reload.enabled: false
    output.elasticsearch:
      hosts: ['elasticsearch.efk.svc.cherry.com:9200']
      index: 'k8s-ds-%{+yyyy.MM.dd}'

    setup.ilm.enabled: false
    setup.template.name: "k8s-ds"
    setup.template.pattern: "k8s-ds*"
    setup.template.overwrite: false
    setup.template.settings:
      index.number_of_shards: 5
      index.number_of_replicas: 0
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-inputs
data:
  kubernetes.yml: |-
    - type: container
      stream: all
      paths: 
        - '/var/lib/docker/containers/*/*.log'
      processors:
        - add_kubernetes_metadata:
            in_cluster: true

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: filebeat
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: filebeat
subjects:
- kind: ServiceAccount
  name: filebeat
  namespace: default
roleRef:
  kind: ClusterRole
  name: filebeat
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: filebeat
  labels:
    k8s-app: filebeat
rules:
- apiGroups: [""]
  resources:
  - namespaces
  - pods
  - nodes
  verbs:
  - get
  - watch
  - list

---

apiVersion: apps/v1 
kind: DaemonSet
metadata:
  name: filebeat
spec:
  selector:
    matchLabels:
      k8s-app: filebeat
  template:
    metadata:
      labels:
        k8s-app: filebeat
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
        operator: Exists
      serviceAccountName: filebeat
      terminationGracePeriodSeconds: 30
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:7.17.22
        args: [
          "-c", "/etc/filebeat.yml",
          "-e",
        ]
        securityContext:
          runAsUser: 0
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: config
          mountPath: /etc/filebeat.yml
          readOnly: true
          subPath: filebeat.yml
        - name: inputs
          mountPath: /usr/share/filebeat/inputs.d
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: config
        configMap:
          defaultMode: 0600
          name: filebeat-config
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: inputs
        configMap:
          defaultMode: 0600
          name: filebeat-inputs
```

![img](https://gitee.com/ljh00928/csdn/raw/master/img/d234b186600f46cab91dfb4d3427d037.png)

