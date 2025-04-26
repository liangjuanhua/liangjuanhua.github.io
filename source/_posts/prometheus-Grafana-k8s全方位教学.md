---
title: prometheus+Grafana+k8s全方位教学
date: 2025-04-16 16:34:12
tags: 监控篇  
categories: 监控篇
---
## 1、架构图

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/39b97ead427144628582592e0c502bcd.png)

## 2、组件

**Prometheus Server**
用于收集和存储时间序列数据。Prometheus Server 是 Prometheus 组件中的核心部分，负责实现对监控数据的获取，存储以及查询。 Prometheus Server 可以通过静态配置管理监控目标，也可以配合使用 Service Discovery 的方式动态管理监控目标，并从这些监控目标中获取数据。其次 Prometheus Server 需要对采集到的监控数据进行存储，Prometheus Server 本身就是一个时序数据库，将采集到的监控数据按照时间序列的方式存储在本地磁盘当中。最后Prometheus Server 对外提供了自定义的 PromQL 语言，实现对数据的查询以及分析。

**Exporter**
用于暴露已有的第三方服务的 metrics 给 Prometheus。Exporter 将监控数据采集的端点通过 HTTP 服务的形式暴露给 Prometheus Server，Prometheus Server 通过访问该 Exporter 提供的 Endpoint 端点，即可获取到需要采集的监控数据。

**Push Gateway**
主要用于短期的 jobs。由于这类 jobs 存在时间较短，可能在 Prometheus 来 pull 之前就消失了。为此，这些 jobs 可以直接向 Prometheus server 端推送它们的 metrics。

**Grafana**
第三方展示工具，可以编写 PromQL 查询语句，通过 http 协议与 prometheus 集成。

**AlertManager**
从 Prometheus Server 端接收到 alerts 后，会进行去除重复数据，分组，并路由到对方的接受方式，发出报警。常见的接收方式有：电子邮件，钉钉、企业微信，pagerduty等。

**Client Library**
客户端库，为需要监控的服务生成相应的 metrics 并暴露给 Prometheus Server。当 Prometheus Server 来 pull 时，直接返回实时状态的 metrics。

  	监控流程：
  		1.exporter节点暴露监控指标;
  		2.Prometheus server修改配置文件监控暴露节点;
  		3.重载配置检查WebUI;
  		4.grafana出图展示;

## 3、prometheus部署

```bash
下载Prometheus的软件包
[root@prometheus-server31 ~]# wget https://github.com/prometheus/prometheus/releases/download/v2.53.2/prometheus-2.53.2.linux-amd64.tar.gz

上传普罗米修斯部署脚本（需要脚本可后台留言~）
[root@prometheus-server31 ~]# tar xf install-prometheus-server.tar.gz 

安装
[root@prometheus-server31 ~]# ./install-prometheus-server.sh i
```

## 4、node-exporter环境搭建

```bash
下载软件包 
[root@prometheus-node41 ~]# wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz

脚本一键部署node-exporter（需要脚本后台留言~）
[root@prometheus-node41 ~]# tar xf install-node-exporter.tar.gz 

安装服务 
[root@prometheus-node41 ~]# ./install-node-exporter.sh i
```

## 5、Prometheus server监控node-exporter实战 

```bash
1.修改Prometheus server的配置文件监控node-exporter节点 
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
...
# 修改通用全局配置
global:
  # Prometheus server抓取数据的间隔时间，默认值为1分钟
  scrape_interval: 3s 
  
...
# 定义抓取配置
scrape_configs:
  	...(添加如下信息)
	
    # 自定义任务的名称
  - job_name: node-exporters
    # 指定采集指标时访问的路径
    metrics_path: /metrics
    # 指定采集指标时使用的协议
    scheme: http
    # 指定被监控的node-exporter节点列表
    static_configs:
    - targets:
      - 10.0.0.41:9100
      - 10.0.0.42:9100
      - 10.0.0.43:9100


2.检查配置文件语法是否正确 
[root@prometheus-server31 ~]# /softwares/prometheus-2.53.2.linux-amd64/promtool check config  /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 

3.Prometheus server加载配置文件
[root@prometheus-server31 ~]# curl -X POST 10.0.0.31:9090/-/reload

4.给检查和加载配置文件起别名
[root@prometheus-server31 ~]# vim ~/.bashrc 
...
alias rr='curl -X POST 10.0.0.31:9090/-/reload'
alias check='/softwares/prometheus-2.53.2.linux-amd64/promtool check config  /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml'

[root@prometheus-server31 ~]# source  ~/.bashrc 

5.查看Prometheus的WebUI验证节点是否加入成功
http://10.0.0.31:9090/targets

6..查看Prometheus的指标数据
node_cpu_seconds_total
```

## 6、PQL

prometheus监控中采集过来的数据统一称为Metrics数据，其并不是代表具体的数据格式，而是一种统计度量计算单位。

当我们需要为某个系统或者某个服务做监控时，就需要使用到metrics。

```bash
前提条件: (所有节点时区同步)
ln -svf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime 
```

#### **体验promql**

```bash
1 查看某个特定的key
node_cpu_seconds_total

2 查看某个节点的指标
node_cpu_seconds_total{instance="10.0.0.41:9100"}

3 查看某个节点的某刻CPU的某种状态
node_cpu_seconds_total{instance="10.0.0.41:9100",cpu="0",mode="idle"}

4 查询最近10s内某个节点CPU的某种状态时间
node_cpu_seconds_total{instance="10.0.0.41:9100",cpu="0",mode="idle"}[10s]

5 统计1分钟内，使用标签过滤器查看"10.0.0.42:9100"节点的第0颗CPU，非空闲状态使用的总时间
node_cpu_seconds_total{mode!="idle",cpu="0", instance="10.0.0.42:9100"}[1m]

6 统计1分钟内，使用标签过滤器查看"10.0.0.42:9100"节点的第0颗CPU，mode名称以字母"i"开头的所有CPU核心。
node_cpu_seconds_total{mode=~"i.*",cpu="0", instance="10.0.0.42:9100"}[1m]

7 统计1分钟内，使用标签过滤器查看"10.0.0.42:9100"节点的第0颗CPU，mode名称不是以字母"i"开头的所有CPU核心。
node_cpu_seconds_total{mode!~"i.*",cpu="0", instance="10.0.0.42:9100"}[1m]

```

#### **Prometheus常用的函数**

```bash
1 increase函数: 用来针对counter数据类型，截取其中一段时间总的增量。		
举个例子:
   increase(node_cpu_seconds_total{mode="idle",cpu="0", instance="10.0.0.42:9100"}[1m])
		统计1分钟内，使用标签过滤器查看"10.0.0.42:9100"节点的第0颗CPU，空闲状态使用的总时间增量。

2 sum函数: 加和的作用。
举个例子:
 sum(increase(node_cpu_seconds_total{mode="idle",cpu="0"}[1m]))
		统计1分钟内，使用标签过滤器查看所有节点的第0颗CPU，空闲状态使用的总时间增量，并将返回结果累加。

3 by函数: 将数据进行分组，类似于MySQL的"GROUP BY"。
举个例子:
	sum(increase(node_cpu_seconds_total{mode="idle"}[1m])) by (instance)
		统计1分钟内，使用标签过滤器查看CPU空闲状态，并将结果进行累加，基于instance进行分组。

4 rate函数: 它的功能是按照设置的时间段，取counter在这个时间段中平均每秒的增量。
举个例子:
	rate(node_cpu_seconds_total{mode="idle",cpu="0", instance="10.0.0.42:9100"}[1m])
		统计1分钟内，使用标签过滤器查看"10.0.0.42:9100"节点的第0颗CPU，空闲状态使用的每秒的增量。
		
increase和rate如何选择:
	(1)对于采集数据频率较低的场景建议使用increase函数，因为使用rate函数可能会出现断点,比如针对硬盘容量监控。
	(2)对于采集数据频率较高的场景建议使用rate函数，比如针对CPU，内存，网络流量等都是可以基于rate函数来采集等。
	
5 topk函数: 取前几位的最高值，实际使用的时候一般会用该函数进行瞬时报警，而不是为了观察曲线图。
举个例子:
	topk(3, rate(node_cpu_seconds_total{mode="idle"}[1m]))
		统计1分钟内，使用标签过滤器查看CPU，所有状态使用的每秒的增量，只查看前3个节点。

6 count函数:
	把数值符合条件的，输出数目进行累加加和。
	比如说企业中有100台服务器，如果只有10台服务器CPU使用率高于80%时候是不需要报警的，但是数量操作70台时就需要报警了。
	
举个例子:
	count(tcp_wait_conn > 500):
		假设(tcp_wait_conn是咱们自定义的KEY。
		若TCP等待数量大于500的机器数量就判断条件为真。

	count(rate(node_cpu_seconds_total{cpu="0",mode="idle"}[1m]))
		对统计的结果进行计数。

7 其他函数  https://prometheus.io/docs/prometheus/latest/querying/functions/	
```

#### 监控CPU的使用情况案例

```bash
1 统计各个节点CPU的使用率
		1.1 我们需要先找到CPU相关的KEY
node_cpu_seconds_total

		1.2 过滤出CPU的空闲时间({mode='idle'})和全部CPU的时间('{}')
node_cpu_seconds_total{mode='idle'}
	过滤CPU的空闲时间。
	
node_cpu_seconds_total{}
	此处的'{}'可以不写，因为里面没有任何参数，代表获取CPU的所有状态时间。
	
		1.3 统计1分钟内CPU的增量时间
increase(node_cpu_seconds_total{mode='idle'}[1m])
	统计1分钟内CPU空闲状态的增量。
	
increase(node_cpu_seconds_total[1m])
	统计1分钟内CPU所有状态的增量。
	
		1.4 将结果进行加和统计
sum(increase(node_cpu_seconds_total{mode='idle'}[1m]))
	将1分钟内所有CPU空闲时间的增量进行加和计算。
	
sum(increase(node_cpu_seconds_total[1m]))
	将1分钟内所有CPU空闲时间的增量进行加和计算。
	
		1.5 按照不同节点进行分组
sum(increase(node_cpu_seconds_total{mode='idle'}[1m])) by (instance)
	将1分钟内所有CPU空闲时间的增量进行加和计算，并按照机器实例进行分组。
	
sum(increase(node_cpu_seconds_total[1m])) by (instance)
	将1分钟内所有CPU空闲时间的增量进行加和计算，并按照机器实例进行分组。
	
		1.6 计算1分钟内CPU空闲时间的百分比
sum(increase(node_cpu_seconds_total{mode='idle'}[1m])) by (instance) / sum(increase(node_cpu_seconds_total[1m])) by (instance)

		1.7 统计1分钟内CPU的使用率，计算公式: (1 - CPU空闲时间的百分比) * 100%。
(1 - sum(increase(node_cpu_seconds_total{mode='idle'}[1m])) by (instance) / sum(increase(node_cpu_seconds_total[1m])) by (instance)) * 100

		1.7 统计1小时内CPU的使用率，计算公式: (1 - CPU空闲时间的百分比) * 100%。
(1 - sum(increase(node_cpu_seconds_total{mode='idle'}[1h])) by (instance) / sum(increase(node_cpu_seconds_total[1h])) by (instance)) * 100


2 计算CPU用户态的1分钟内百分比
sum(increase(node_cpu_seconds_total{mode='user'}[1m])) by (instance) / sum(increase(node_cpu_seconds_total[1m])) by (instance) * 100

3 计算CPU内核态的1分钟内百分比
(sum(increase(node_cpu_seconds_total{mode='system'}[1m])) by (instance) / sum(increase(node_cpu_seconds_total[1m])) by (instance)) * 100

4 计算CPU IO等待时间的1分钟内百分比
(sum(increase(node_cpu_seconds_total{mode='iowait'}[1m])) by (instance) / sum(increase(node_cpu_seconds_total[1m])) by (instance)) * 100
```

## 7、grafana

#### grafana部署

```bash
1. 下载grafana
wget https://dl.grafana.com/enterprise/release/grafana-enterprise_11.1.4_amd64.deb

2.安装grafana
[root@prometheus-server31 ~]# apt-get install -y adduser libfontconfig1 musl
[root@prometheus-server31 ~]# 
[root@prometheus-server31 ~]# dpkg -i grafana-enterprise_11.1.4_amd64.deb

3.启动grafana 
[root@prometheus-server31 ~]# systemctl enable --now grafana-server
[root@prometheus-server31 ~]# 
[root@prometheus-server31 ~]# ss -ntl | grep 3000
LISTEN 0      4096               *:3000            *:*    

4.访问grafana的WebUI
http://10.0.0.31:3000/login
- 1.初始化的用户名和密码均为: admin 
```

**配置Prometheus的数据源**

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/781412229a9249699bcf56555933b72f.png)


![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/89dcebbc6a794cdd9ce03c819b5ff35c.png)

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/45133fff7e4146bf9b3b4a7e6b975270.png)


**添加服务端地址**
![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/69aff8fdd139453da62a7f029ce5fdb1.png)
![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/96440f22bc744cf98ef6b9a796e7e8e6.png)


**导入样板**
![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/849314b2241045a28049f184cabcafcd.png)


**选择样板id**
![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/32037fdf1cc2467fa22574fc36677118.png)



**选择数据源**

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/2218c681c0de4de39e80bc63a92c4ed0.png)



**配置grafana展示node-exporter数据**
![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/2aee5db9f02445a4b4bbebe06cad4744.png)

#### grafana自定义dashboard

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/0af2d0c02ef44fbfab2c39a58387bd87.png)


#### grafana实现备份和恢复

保存json文件，恢复的时候可粘贴内容或者导入文件

## 8、联邦模式

默认情况下，prometheus采集的数据会存储到本地，这意味者prometheus在这种工作模式下，可能会存在单机存储的瓶颈。

- 为了解决prometheus对于数据的采集压力，我们可以采用联邦模式来部署prometheus

- 所谓联邦模式就是部署多个server共同采集数据



#### **联邦架构图**

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/26cbfe2310e248c59a3e22826211db12.png)



#### **部署联邦模式**

1.修改prometheus server32配置

```bash
修改prometheus server配置文件
[root@prometheus-server32 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
...
scrape_configs:
  ...

  - job_name: 'file-service-discovery-32'
    static_configs:
    - targets:
      - "10.0.0.41:9100"

重载prometheus server
[root@prometheus-server32 ~]# curl -X POST http://10.0.0.32:9090/-/reload

验证数据是否采集成功
http://10.0.0.32:9090/targets
```

2.修改prometheus server33配置

```bash
修改prometheus server的配置文件
[root@prometheus-server33 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 

...
scrape_configs:
  ...
  - job_name: 'file-service-discovery-33'
    static_configs:
    - targets:
      - "10.0.0.42:9100"
	  - "10.0.0.43:9100"

重载prometheus server
[root@prometheus-server33 ~]# curl -X POST http://10.0.0.33:9090/-/reload

验证数据是否采集成功
http://10.0.0.33:9090/targets
```

3.修改Prometheus server31配置

```bash
修改prometheus server的配置文件
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
...
scrape_configs:
  ...
  - job_name: "prometheus-federate-32"
    metrics_path: "/federate"
    # 用于解决标签的冲突问题，有效值为: true和false，默认值为false
    # 当设置为true时，将保留抓取的标签以忽略服务器自身的标签。说白了会覆盖原有标签。
    # 当设置为false时，则不会覆盖原有标签，而是在标点前加了一个"exported_"前缀。
    honor_labels: true
    params:
       "match[]":
       - '{job="promethues"}'
       - '{__name__=~"job:.*"}'
       - '{__name__=~"node.*"}'
    static_configs:
    - targets:
        - "10.0.0.32:9090"

  - job_name: "prometheus-federate-33"
    metrics_path: "/federate"
    honor_labels: true
    params:
       "match[]":
       - '{job="promethues"}'
       - '{__name__=~"job:.*"}'
       - '{__name__=~"node.*"}'
    static_configs:
    - targets:
        - "10.0.0.33:9090"

检查配置文件语法
[root@prometheus-server31 ~]# check 

重载prometheus server
[root@prometheus-server31 ~]# rr

验证数据是否采集成功
http://10.0.0.31:9090/targets

```



## 9、监控流程

普罗米修斯监控可分为两类，云原生应用和非云原生应用。

云原生应用提供metrics，不需要安装exporters客户端，直接修改配置文件即可

非云原生应用需要安装exportes客户端，并启动客户端，服务端yaml文件加入客户端ip和端口


## 10、监控zookeeper集群

```bash
修改zookeeper集群的配置文件
[root@elk91 ~]# vim /softwares/apache-zookeeper-3.8.4-bin/conf/zoo.cfg 
...
# https://prometheus.io Metrics Exporter
metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
metricsProvider.httpHost=0.0.0.0
metricsProvider.httpPort=7000
metricsProvider.exportJvmInfo=true
...           
[root@elk91 ~]# systemctl restart zk

测试服务是否正常
[root@elk91 ~]# for i in `seq 91 93`; do echo stat | nc 10.0.0.$i 2181 | grep Mode;done
Mode: follower
Mode: leader
Mode: follower

访问webUI
http://10.0.0.91:7000/metrics

Prometheus server配置监控zookeeper集群
[root@prometheus-server31 ~]# tail -6 /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
  - job_name: zookeeper
    static_configs:
    - targets:
      - 10.0.0.91:7000
      - 10.0.0.92:7000
      - 10.0.0.93:7000
[root@prometheus-server31 ~]# 
[root@prometheus-server31 ~]# check
重载服务
[root@prometheus-server31 ~]# rr

访问Prometheus的WebUI进行验证
http://10.0.0.31:9090/targets

grafana导入模板
10465
```



## 11、客户端下载地址

```bash
监控应用的流程Prometheus
https://prometheus.io/docs/instrumenting/exporters/
```



## 12、监控elasticsearch集群

```bash
1.下载elasticsearch exporter
https://github.com/prometheus-community/elasticsearch_exporter/releases/download/v1.7.0/elasticsearch_exporter-1.7.0.linux-amd64.tar.gz

2.解压软件包 
[root@elk91 ~]# tar xf elasticsearch_exporter-1.7.0.linux-amd64.tar.gz 

3.启动测试
[root@elk91 elasticsearch_exporter-1.7.0.linux-amd64]# ./elasticsearch_exporter --es.uri="http://elastic:123456@10.0.0.93:9200" --web.listen-address=:9114 --web.telemetry-path="/metrics" 

4.Prometheus server监控es的exporter
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
...

  - job_name: elasticsearch
    static_configs:
    - targets:
      - 10.0.0.91:9114
      
[root@prometheus-server31 ~]# rr

5.查看Prometheus的WebUI是否监控到目标
http://10.0.0.31:9090/targets

6.grafana出图展示
14191
```



## 13、监控kafka集群

```bash
1.启动kafka集群
[root@elk91 ~]# kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties 

[root@elk92 ~]# kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties 

[root@elk93 ~]# kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties 

2.验证kafka服务是否正常
[root@elk91 ~]# zkCli.sh ls /kafka371/brokers/ids  | grep "^\["

3.下载kafka的exporter
wget https://github.com/danielqsj/kafka_exporter/releases/download/v1.7.0/kafka_exporter-1.7.0.linux-amd64.tar.gz

4.解压目录中指定文件kafka_exporter到指定路径
[root@elk91 ~]# tar xf  kafka_exporter-1.7.0.linux-amd64.tar.gz -C /usr/local/bin/ kafka_exporter-1.7.0.linux-amd64/kafka_exporter  --strip-components=1

5.启动 kafka_exporter
[root@elk91 ~]# kafka_exporter --web.listen-address=":9308" --web.telemetry-path="/metrics"  --kafka.version="3.7.1" --kafka.server=10.0.0.93:9092

6.访问测试kafka的exporter页面
http://10.0.0.91:9308/metrics

7.Prometheus配置监控kafka的exporter
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
...
  - job_name: kafka
    static_configs:
    - targets:
      - 10.0.0.91:9308
  
[root@prometheus-server31 ~]# rr

8.查看Prometheus的WebUI是否监控到目标
http://10.0.0.31:9090/targets

9.grafana出图展示
12460

9.测试验证准确信
		9.1 创建topic
[root@elk93 ~]# kafka-topics.sh --bootstrap-server 10.0.0.91:9092 --create --topic xixi --partitions 3 
Created topic xixi.
[root@elk93 ~]# 

		
		9.2 启动消费者
[root@elk92 ~]# kafka-console-consumer.sh --bootstrap-server 10.0.0.91:9092  --topic xixi 

		9.3 启动生产者
[root@elk93 ~]# kafka-console-producer.sh --bootstrap-server 10.0.0.91:9092  --topic xixi
```



## 14、监控Jenkins服务

```bash
1.jenkins安装Prometheus插件
如果安装插件失败，可以直接导入tar包到/var/lib/jenkins/plugins目录并重启。
tar xf jenkins-plugins.tar.gz 

2.验证Jenkins的metrics组件是否生效
[root@jenkins211 plugins]# systemctl restart jenkins
http://10.0.0.211:8080/prometheus/

3.验证Jenkins的metrics组件是否生效
http://10.0.0.211:8080/prometheus/

4.修改Prometheus的配置文件
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
...
  - job_name: jenkins
    metrics_path: /prometheus
    static_configs:
    - targets:
      - 10.0.0.211:8080
...

[root@prometheus-server31 ~]# rr

5.访问Prometheus的WebUI
http://10.0.0.31:9090/targets

6.导入Jenkins的相关模板
9964
9524
12646
```



## 15、监控mysql服务

```bash
1.部署MySQL
[root@jenkins211 ~]# docker run --name mysql-server -d \
             -e MYSQL_USER="root" \
             -e MYSQL_PASSWORD="123456" \
             -e MYSQL_ALLOW_EMPTY_PASSWORD="yes" \
             --network=host \
			 --restart unless-stopped \
             mysql:8.4.2-oracle \
             --character-set-server=utf8mb4 --collation-server=utf8mb4_bin 
             
[root@jenkins211 ~]# docker ps -l
CONTAINER ID   IMAGE                COMMAND                  CREATED          STATUS          PORTS     NAMES
5db1d0101b5c   mysql:8.3.0-oracle   "docker-entrypoint.s…"   13 seconds ago   Up 13 seconds             mysql-server
[root@jenkins211 ~]# 
[root@jenkins211 ~]# ss -ntl | grep 3306
LISTEN 0      151                *:3306             *:*          
LISTEN 0      70                 *:33060            *:*   


2.下载mysql exporter 
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz

3.安装mysql exporters
[root@jenkins211 ~]# tar xf mysqld_exporter-0.15.1.linux-amd64.tar.gz -C /usr/local/bin/ mysqld_exporter-0.15.1.linux-amd64/mysqld_exporter  --strip-components=1

4.准备MySQL的链接认证文件，创建MySQL的配置，指定默认的用户名和密码
[root@jenkins211 ~]# cat  ~/.my.cnf 
[client]
user=root
password=123456

5.运行mysqld-exporter
[root@jenkins211 ~]# mysqld_exporter --mysqld.address="10.0.0.211:3306" --web.listen-address=:9104 --config.my-cnf="/root/.my.cnf"

6.访问mysqld_exporter的webUI
http://10.0.0.211:9104/metrics

7.修改Prometheus的配置文件
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
...
  - job_name: mysql
    static_configs:
    - targets:
      - 10.0.0.211:9104
      
[root@prometheus-server31 ~]# rr

8.查看Prometheus是否监控到数据
http://10.0.0.31:9090/targets

9.grafana出图展示
18949
17320
14057
```



## 16、监控Redis服务

```bash
1.部署Redis
[root@jenkins211 ~]# docker run -d --name redis-server --restart always --network host  redis:7.2.5

2.下载redis-exporter
wget https://github.com/oliver006/redis_exporter/releases/download/v1.52.0/redis_exporter-v1.52.0.linux-amd64.tar.gz

3.解压软件包到PATH路径
[root@jenkins211 ~]# tar xf redis_exporter-v1.61.0.linux-amd64.tar.gz -C /usr/local/bin/ redis_exporter-v1.61.0.linux-amd64/redis_exporter --strip-components=1
[root@jenkins211 ~]# ll /usr/local/bin/

4.运行redis-exporter
[root@jenkins211 ~]# redis_exporter -web.listen-address=:9121 -web.telemetry-path=/metrics  -redis.addr=redis://10.0.0.211:6379

5.访问redis-exporter的WebUI
http://10.0.0.211:9121/metrics

6.修改Prometheus的配置文件
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
...
  - job_name: redis
    static_configs:
    - targets:
      - 10.0.0.211:9121
[root@prometheus-server31 ~]# rr


7.访问Prometheus的WebUI
http://10.0.0.31:9090/targets
	
8.grafana出图展示
763
14091	
```



## 17、安装grafana插件

**在线安装**

```bash
grafana的版本为9.5.21
软件包下载到/var/lib/grafana/plugins/目录

[root@prometheus-server31 grafana]# grafana-cli plugins install natel-discrete-panel
```

**离线安装**

```bash
[root@grafana71 ~]# wget  https://grafana.com/api/plugins/natel-discrete-panel/versions/latest/download -O /opt/natel-discrete-panel.zip

[root@prometheus-server31 ~]# unzip natel-discrete-panel-0.1.1.zip -d /var/lib/grafana/plugins/

[root@prometheus-server31 ~]# systemctl restart grafana-server

[root@prometheus-server31 ~]# ss -ntl | grep 3000
LISTEN 0      4096               *:3000            *:*          
```



## 18、监控nginx服务

**编译安装nginx**

```bash
1 安装编译工具
CentOS：
yum -y install git wget gcc make zlib-devel gcc-c++ libtool openssl openssl-devel

Ubuntu：
apt -y install git wget gcc make zlib1g-dev build-essential libtool openssl libssl-dev

2 克隆nginx-module-vts模块
git clone git://github.com/vozlt/nginx-module-vts.git
或者
git clone https://gitee.com/jasonyin2020/nginx-module-vts.git

3 下载nginx软件包
wget https://nginx.org/download/nginx-1.26.2.tar.gz

4 解压nginx
tar xf nginx-1.26.2.tar.gz


5 配置nginx
cd nginx-1.26.2
./configure --prefix=/softwares/nginx \
  --with-http_ssl_module \
  --with-http_v2_module \
  --with-http_realip_module \
  --without-http_rewrite_module \
  --with-http_stub_status_module \
  --without-http_gzip_module  \
  --with-file-aio \
  --with-stream \
  --with-stream_ssl_module \
  --with-stream_realip_module \
  --add-module=/root/nginx-module-vts

6 编译并安装nginx
make -j 2 && make install

7 修改nginx的配置文件
vim /softwares/nginx/conf/nginx.conf
...
http {
    #加入编译的status模块，将请求代理到31:9090端口
    vhost_traffic_status_zone;
    upstream promethues {
       server 10.0.0.31:9090;
    }
    ...
    server {
        ...
        location / {
            root   html;
            # index  index.html index.htm;
            proxy_pass http://promethues;
        }

        location /status {
            vhost_traffic_status_display;
            vhost_traffic_status_display_format html;
        }
    }
}

8 检查配置文件语法
[root@jenkins211 ~]# /softwares/nginx/sbin/nginx -t

9 启动nginx
[root@jenkins211 ~]# /softwares/nginx/sbin/nginx
[root@jenkins211 ~]# 
[root@jenkins211 ~]# ss -ntl | grep 80
LISTEN 0      511          0.0.0.0:80        0.0.0.0:*      

10 访问nginx的状态页面
http://10.0.0.211/status/format/prometheus
http://10.0.0.211/status
```

**安装nginx-vtx-exporter**

```bash
1.下载nginx-vtx-exporter,不建议下载最新版本
wget https://github.com/hnlq715/nginx-vts-exporter/releases/download/v0.10.3/nginx-vts-exporter-0.10.3.linux-amd64.tar.gz

2 解压软件包到path路径
tar xf nginx-vts-exporter-0.10.3.linux-amd64.tar.gz -C /usr/local/bin/ nginx-vts-exporter-0.10.3.linux-amd64/nginx-vts-exporter --strip-components=1

3 运行nginx-vtx-exporter
[root@jenkins211 ~]# nginx-vts-exporter -nginx.scrape_uri=http://10.0.0.211/status/format/json
```

**配置prometheus采集nginx数据**

```bash
1 修改配置文件
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 

...
scrape_configs:
  ...
  - job_name: "nginx-exporter"
    metrics_path: "/status/format/prometheus"
    static_configs:
      - targets:
          - "10.0.0.211:80"

  - job_name: "nginx-vts-exporter"
    static_configs:
      - targets:
          - "10.0.0.211:9913"
          
[root@prometheus-server31 ~]# rr

2 访问Prometheus的WebUI
http://10.0.0.31:9090/targets

3 导入grafana模板
2949
```



## 19、监控tomcat服务

```bash
1 基于Dockerfile构建tomcat-exporter
[root@jenkins211 ~]# git clone https://gitee.com/jasonyin2020/tomcat-exporter.git

2.编译镜像
[root@jenkins211 ~]# cd tomcat-exporter/
[root@jenkins211 tomcat-exporter]# chmod +x build.sh 
[root@jenkins211 tomcat-exporter]# ./build.sh 

3 运行tomcat镜像
[root@jenkins211 ~]# docker run -dp 18080:8080 --name tomcat-server registry.cn-hangzhou.aliyuncs.com/yinzhengjie-k8s/tomcat9-app:v1

4.访问tomcat应用
http://10.0.0.211:18080/metrics/
http://10.0.0.211:18080/myapp/ 

5.配置prometheus监控tomcat应用
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
...
scrape_configs:
  ...
  - job_name: "tomcat-exporter"
    static_configs:
      - targets: 
          - "10.0.0.211:18080"
          
5.2 导入grafana模板 
https://github.com/nlighten/tomcat_exporter/blob/master/dashboard/example.json
```



## 20、监控容器cadvisor

它是一个正在运行的守护进程，用于收集、聚合、处理和导出有关正在运行的容器的信息。

```bash
官网地址：https://github.com/google/cadvisor

导入镜像
[root@jenkins211 ~]# docker load -i cadvisor-amd64-0.49.1.tar.gz 

运行容器
[root@jenkins211 ~]# 
VERSION=v0.49.1 
docker run \
  -v /:/rootfs:ro \
  -v /var/run:/var/run:ro \
  -v /sys:/sys:ro \
  -v /var/lib/docker/:/var/lib/docker:ro \
  -v /dev/disk/:/dev/disk:ro \
  -p 28080:8080 \
  -d \
  --name=cadvisor \
  --privileged \
  --device=/dev/kmsg \
  gcr.io/cadvisor/cadvisor-amd64:$VERSION
54149a621e6bcd9a612fc0b3c755eea91d7466b52bf732a92816c22993b2d635

prometheus采集cAdvisor容器
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml
...
scrape_configs:
  ...
  - job_name: "prometheus-cAdvisor"
    static_configs:
    - targets:
        - "10.0.0.211:28080"
        
[root@prometheus-server31 ~]# rr
10.0.0.211:28080/metrics

访问WebUI验证配置是否生效
http://10.0.0.31:9090/targets

导入grafana模板
315  
10619


grafana的官方优化思路-对于容器出现小数的情况
针对10619模板，当容器数量增多时，如果容器出现小数点，微调即可。

Value options  --->  "Last*"
```

 

## 21、基于docker部署Prometheus相关组件

```bash
1.部署Prometheus -server 
[root@jenkins211 ~]# docker run -d --network host --name prometheus-server prom/prometheus:v2.53.2 

2.部署node-exporter
[root@jenkins211 ~]# docker run  -d --name node-exporter --network host  prom/node-exporter:v1.8.2 

3.配置Prometheus server监控node-exporter
修改配置文件
[root@jenkins211 ~]# docker exec -it prometheus-server sh
/prometheus $ 
/prometheus $ vi /etc/prometheus/prometheus.yml 
...
                                         
  - job_name: "prometheus-node-exporter" 
    static_configs:                     
    - targets:                          
      - 10.0.0.211:9100
      
重新加载配置
[root@jenkins211 ~]# docker restart prometheus-server 
验证是否加载成功
http://10.0.0.211:9090/targets

4.部署grafana组件
部署 
[root@jenkins211 ~]# docker run -d --name grafana --network host grafana/grafana:9.5.21 
访问测试 
http://10.0.0.211:3000


5.部署pushgateway组件
部署 
[root@jenkins211 ~]# docker run -d --name pushgateway --network host prom/pushgateway:v1.9.0 
访问测试 
http://10.0.0.211:9091/

6.部署alertmanager组件
部署
[root@jenkins211 ~]# docker run -d --name alertmanager --network host prom/alertmanager:v0.27.0 
访问测试 
http://10.0.0.211:9093/#/alerts
```



## 22、文件发现服务

静态配置：之前使用的都是静态分析，每次都要重启服务或者热加载文件

动态配置：可以动态发现服务，无需热加载文件

动态配置可分为json文件和yaml文件

```bash
1.修改prometheus的配置文件 
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 

...
  - job_name: 'file-service-discovery-json'
    # 基于文件的服务发现为动态发现
    file_sd_configs:
        # 指定文件路径
      - files:
          - /softwares/prometheus-2.53.2.linux-amd64/config/*.json

  - job_name: 'file-service-discovery-yaml'
    file_sd_configs:
      - files:
          - /softwares/prometheus-2.53.2.linux-amd64/config/*.yaml
          
          
2.重新加载配置 
[root@prometheus-server31 ~]# rr

3.访问WebUI验证配置是否生效
http://10.0.0.31:9090/config

4.创建配置文件模拟基于动态的监控
创建目录 
[root@prometheus-server31 ~]# mkdir -pv /softwares/prometheus-2.53.2.linux-amd64/config
创建json文件监控41节点
[root@prometheus-server31 ~]# cat /softwares/prometheus-2.53.2.linux-amd64/config/linux.json
[
  {
    "targets": [ "10.0.0.41:9100" ],
    "labels": {
      "school": "cherry",
      "class": "123456"
    }
  }
]
创建yaml文件监控42和43节点
[root@prometheus-server31 ~]# cat /softwares/prometheus-2.53.2.linux-amd64/config/haha.yaml 
- targets:
    - '10.0.0.42:9100'
    - '10.0.0.43:9100'
  labels:
    apps: yaml
    address: shahe
    
再次查看Prometheus的WebUI
http://10.0.0.31:9090/targets

参考链接:	https://prometheus.io/docs/prometheus/2.53/configuration/configuration/#file_sd_config
```





## **23、consul服务发现**

普罗米修斯服务端不能直接发现node节点，由consul服务端将node节点告诉过普罗米修斯服务端，consul也属于动态发现服务

**node节点部署consul集群**

```bash
1 下载consul
wget https://releases.hashicorp.com/consul/1.19.1/consul_1.19.1_freebsd_amd64.zip

2 快速部署consul集群
下载consul
wget https://releases.hashicorp.com/consul/1.19.1/consul_1.19.1_freebsd_amd64.zip

解压consul
unzip consul_1.19.1_linux_amd64.zip  -d /usr/local/bin/

运行consul 集群
leader43:
consul agent -server -bootstrap -bind=10.0.0.43 -data-dir=/softwares/consul -client=10.0.0.43 -ui


follower42:
consul agent  -bind=10.0.0.42 -data-dir=/softwares/consul -client=10.0.0.42 -ui -retry-join=10.0.0.43


follower41:
consul agent -server -bind=10.0.0.41 -data-dir=/softwares/consul -client=10.0.0.41 -ui -retry-join=10.0.0.43

访问console服务的WebUI，查看node节点
http://10.0.0.43:8500/ui/dc1/nodes
```

**配置自动发现**

```bash
1 修改prometheus的配置文件
vim /softwares/prometheus/prometheus.yml
...
scrape_configs:
  ...
  - job_name: "consul-seriver-discovery"
    # 配置基于consul的服务发现
    consul_sd_configs:
        # 指定consul的服务器地址，若不指定，则默认值为"localhost:8500".
      - server: 10.0.0.43:8500
      - server: 10.0.0.42:8500
      - server: 10.0.0.41:8500
    relabel_configs:
        # 匹配consul的源标签字段，表示服务名称
      - source_labels: [__meta_consul_service]
        # 指定源标签的正则表达式，若不定义，默认值为"(.*)"
        regex: consul
        # 执行动作为删除，默认值为"replace",有效值有多种
        #   https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_action
        action: drop

2 检查配置文件是否正确并重新加载配置
[root@prometheus-server31 ~]# check 
[root@prometheus-server31 ~]# rr

3.被监控节点注册到console集群
注册节点
[root@prometheus-server31 prometheus-2.53.2.linux-amd64]# curl -X PUT -d '{"id":"prometheus-node42","name":"prometheus-node42","address":"10.0.0.42","port":9100,"tags":["node-exporter"],"checks": [{"http":"http://10.0.0.42:9100","interval":"5m"}]}' http://10.0.0.42:8500/v1/agent/service/register

[root@prometheus-server31 prometheus-2.53.2.linux-amd64]# curl -X PUT -d '{"id":"prometheus-node41","name":"prometheus-node42","address":"10.0.0.41","port":9100,"tags":["node-exporter"],"checks": [{"http":"http://10.0.0.41:9100","interval":"5m"}]}' http://10.0.0.42:8500/v1/agent/service/register

[root@prometheus-server31 prometheus-2.53.2.linux-amd64]# curl -X PUT -d '{"id":"prometheus-node43","name":"prometheus-node42","address":"10.0.0.43","port":9100,"tags":["node-exporter"],"checks": [{"http":"http://10.0.0.43:9100","interval":"5m"}]}' http://10.0.0.42:8500/v1/agent/service/register

注销节点,在哪个节点注册就要在哪个节点注销
[root@prometheus-server31 prometheus-2.53.2.linux-amd64]# curl -X PUT http://10.0.0.42:8500/v1/agent/service/deregister/prometheus-node42

[root@prometheus-server31 prometheus-2.53.2.linux-amd64]# curl -X PUT http://10.0.0.42:8500/v1/agent/service/deregister/prometheus-node41

[root@prometheus-server31 prometheus-2.53.2.linux-amd64]# curl -X PUT http://10.0.0.42:8500/v1/agent/service/deregister/prometheus-node43


```



## 24、pushgateway自定义监控指标 

- Prometheus 采用 pull 模式，可能由于不在一个子网或者防火墙原因，导致 Prometheus 无法直接拉取各个 target 数据。
- 在监控业务数据的时候，需要将不同数据汇总, 由 Prometheus 统一收集。

```bash
1.下载组件
wget https://github.com/prometheus/pushgateway/releases/download/v1.9.0/pushgateway-1.9.0.linux-amd64.tar.gz

2.解压软件包 
[root@prometheus-server32 ~]# tar xf pushgateway-1.9.0.linux-amd64.tar.gz  -C /softwares/

3.启动pushgateway组件 
[root@prometheus-server32 ~]# cd /softwares/pushgateway-1.9.0.linux-amd64/
[root@prometheus-server32 pushgateway-1.9.0.linux-amd64]# ./pushgateway 

4.访问pushgateway的WebUI
http://10.0.0.32:9091/#

5.Prometheus server监控pushgateway 
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
...
  - job_name: pushgateway
    # 若不指定则默认值为false。
    # 当设置为true时，若采集的指标包含中和内置的标签冲突时(比如job,instance)会覆盖。
    # 当设置为false时，则不会覆盖，而是在标签前面加一个"exported_*"字段。
    honor_labels: true
    static_configs:
    - targets:
      - 10.0.0.32:9091
      
[root@prometheus-server31 ~]# rr

7.推送数据到pushgateway组件
-------------------------
传递的数据是键值对，KEY一般是字符串类型，而value必须是一个数字类型。
[root@prometheus-server31 ~]# echo "student_online 97" | curl --data-binary @-  http://10.0.0.32:9091/metrics/job/student/instance/10.0.0.31

8.在Prometheus的WebUI验证数据是否推送成功
在Prometheus的WebUI中搜索“student_online”
```



## 25、prometheus监控tcp的12种状态案例

**查看单个状态脚本**

```bash
[root@prometheus-server31 ~]# cat /usr/local/bin/tcp_status.sh  
#!/bin/bash
# auther: cherry
# school: 001
# class: 002
# office: www.cherry.com


# 定义TCP的12种状态
ESTABLISHED_COUNT=0
SYN_SENT_COUNT=0
SYN_RECV_COUNT=0
FIN_WAIT1_COUNT=0
FIN_WAIT2_COUNT=0
TIME_WAIT_COUNT=0
CLOSE_COUNT=0
CLOSE_WAIT_COUNT=0
LAST_ACK_COUNT=0
LISTEN_COUNT=0
CLOSING_COUNT=0
UNKNOWN_COUNT=0

# 定义任务名称
JOB_NAME=tcp_status
# 定义实例名称
INSTANCE_NAME=prometheus32
# 定义pushgateway主机
HOST=10.0.0.32
# 定义pushgateway端口
PORT=9091

# TCP的12种状态
ALL_STATUS=(ESTABLISHED SYN_SENT SYN_RECV FIN_WAIT1 FIN_WAIT2 TIME_WAIT CLOSE CLOSE_WAIT LAST_ACK LISTEN CLOSING UNKNOWN)

# 声明一个关联数组,类似于py的dict,go的map
declare -A tcp_status

# 统计TCP的12种状态
for i in ${ALL_STATUS[@]}
do
  temp=`netstat -untalp | grep $i  | wc -l`
  tcp_status[${i}]=$temp
done

# 将统计后的结果发送到pushgateway
for i in ${!tcp_status[@]}
do 
   data="$i ${tcp_status[$i]}"
   # TODO: shell如果想要设计成相同key不同标签的方式存在问题，只会有最后一种状态被发送
   # 目前我怀疑是pushgateway组件不支持同一个metrics中key所对应的value不同的情况。
   #data="tcp_all_status{status=\"$i\"} ${tcp_status[$i]}"
   #echo $data
   echo $data | curl --data-binary @-  http://${HOST}:${PORT}/metrics/job/${JOB_NAME}/instance/${INSTANCE_NAME}
   # sleep 1
done

查看pushgateway
http://10.0.0.32:9091/#

普罗米修斯查看是否监测
ESTABLISHED.........

```

**查看多个状态值**

```bash
[root@prometheus-server31 ~]# cat -A /usr/local/bin/tcp_status2.sh
#!/bin/bash$
$
# M-hM-.M->M-gM-=M-. Pushgateway M-gM-^ZM-^D URL$
pushgateway_url="http://10.0.0.42:9091/metrics/job/tcp_status"$
time=$(date +%Y-%m-%d+%H:%M:%S)$
$
state="SYN-SENT SYN-RECV FIN-WAIT-1 FIN-WAIT-2 TIME-WAIT CLOSE CLOSE-WAIT LAST-ACK LISTEN CLOSING ESTAB"$
for i in  $state$
 do$
 t=`ss -tan |grep $i |wc -l`$
 echo tcp_connections{state=\""$i"\"} $t >>/tmp/tcp.txt$
done;$
$
cat /tmp/tcp.txt | curl --data-binary @- $pushgateway_url$
rm -rf  /tmp/tcp.txt$
```



## 26、黑盒监控服务

黑盒监控服务也属于自定义的一种监控指标。

1.所谓的黑盒监控
黑盒监控指的是事故已经发生了，才监控到，表示的是从外部监控。举例例子: 网站挂了。

白盒监控指的是服务内部暴露出来的指标，可以更早的预判出问题可能发生的点。举例例子: 当前服务器的负载，队列等待处理的数量异常过高。



2.blackbox_exporter概述
blackbox exporter支持基于HTTP, HTTPS, DNS, TCP, ICMP, gRPC协议来对目标节点进行监控。

比如基于http协议我们可以探测一个网站的返回状态码为200判读服务是否正常。

比如基于TCP协议我们可以探测一个主机端口是否监听。

比如基于ICMP协议来ping一个主机的连通性。

比如基于gRPC协议来调用接口并验证服务是否正常工作。

比如基于DNS协议可以来检测域名解析。



**部署blackbox exporter**

```bash
下载blackbox exporter
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz

解压软件包
[root@prometheus-server32 ~]# tar xf blackbox_exporter-0.25.0.linux-amd64.tar.gz -C  /softwares/

启动服务
[root@prometheus-server32 ~]# cd /softwares/blackbox_exporter-0.25.0.linux-amd64/
[root@prometheus-server32 blackbox_exporter-0.25.0.linux-amd64]# ./blackbox_exporter

访问blackbox的WebUI
http://10.0.0.32:9115/metrics
```



#### 基于blackbox的http模块监控网站状态

```bash
修改Prometheus配置文件
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 

...
scrape_configs:
  ...
    # 指定作业的名称，生成环境中，通常是指一类业务的分组配置。
  - job_name: 'blackbox-exporter-http'
    # 修改访问路径，若不修改，默认值为"/metrics"
    metrics_path: /probe
    # 配置URL的相关参数
    params:
      # 此处表示使用的是blackbox的http模块，从而判断相应的返回状态码是否为200
      module: [http_2xx] 
	  # 下面这两个标签是我自定义的，便于大家理解
      school: [001]
      class: ["002"]
    # 静态配置，需要手动指定监控目标
    static_configs:
        # 需要监控的目标
      - targets:
          # 支持https协议
        - https://www.cherry.com/
          # 支持http协议
        - http://10.0.0.41
          # 支持http协议和自定义端口
        - http://10.0.0.31:9090
    # 对目标节点进行重新打标签配置
    relabel_configs:
        # 指定源标签，此处的"__address__"表示内置的标签，存储的是被监控目标的IP地址
      - source_labels: [__address__]
        # 指定目标标签，其实就是在"Endpoint"中加了一个target字段(用于指定监控目标)，
        target_label: __param_target
        # 指定需要执行的动作，默认值为"replace"，常用的动作有: replace, keep, and drop。
        # 但官方支持十几种动作： https://prometheus.io/docs/prometheus/2.53/configuration/configuration/#relabel_action
        # 将"__address__"传递给target字段。
        action: replace
      - source_labels: [__param_target]
        target_label: instance
        #target_label: instance2024
        
        # 上面的2个配置段也可以改写成如下的配置哟~
     # - source_labels: [__address__]
     #   target_label: instance
     #   action: replace
     # - source_labels: [instance]
     #   target_label: __param_target
     #   action: replace
      - target_label: __address__
        # 指定要替换的值,此处我指定为blackbox exporter的主机地址
        replacement: 10.0.0.32:9115

检查配置文件是否正确并重新加载配置文件
[root@prometheus-server31 ~]# check 
[root@prometheus-server31 ~]# rr

访问prometheus的WebUI
http://10.0.0.31:9090/targets

访问blackbox exporter的WebUI
http://10.0.0.41:9115/

grafana展示数据
7587
13659
```

#### 基于blackbox的ICMP监控目标主机是否存活

```bash
1 修改Prometheus配置文件
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
...
scrape_configs:
  ...
  - job_name: 'blackbox-exporter-icmp'
    metrics_path: /probe
    params:
      # 如果不指定模块，则默认类型为"http_2xx"，不能乱写!乱写监控不到服务啦!
      module: [icmp]
    static_configs:
      - targets:
          - 10.0.0.41
          - 10.0.0.42
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        # 指定注意的是，如果instance不修改，则instance和"__address__"的值相同
        # target_label: ip
        target_label: instance
      - target_label: __address__
        replacement: 10.0.0.32:9115 
        
2 检查配置文件是否正确并重新加载配置
[root@prometheus-server31 ~]# rr

3 访问prometheus的WebUI
http://10.0.0.31:9090/targets

4.访问blackbox的WebUI
http://10.0.0.32:9115/

5.grafana过滤jobs数据
基于"blackbox-exporter-icmp"标签进行过滤。
```

#### 基于blackbox的TCP案例监控服务存活

```bash
1 修改Prometheus配置文件
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
 
...
scrape_configs:
  ...
  - job_name: 'blackox-exporter-tcp'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets:
          - 10.0.0.41:80
          - 10.0.0.42:22
          - 10.0.0.31:9090
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 10.0.0.32:9115
        
2 检查配置文件是否正确并重新加载配置文件
[root@prometheus-server31 ~]# rr

3 访问prometheus的WebUI
http://10.0.0.31:9090/targets

4.访问blackbox exporter的WebUI
http://10.0.0.32:9115/

5.使用grafana查看数据
基于"blackbox-exporter-tcp"标签进行过滤。
```



## 27、远端存储VictoriaMetrics

VictoriaMetrics是一个快速、经济高效且可扩展的监控解决方案和时间序列数据库。

普罗米修斯可以将数据远程存储到VictoriaMetrics。默认情况下，普罗米修斯数据存储于本地文件的 TSDB 中，不利于容灾和做数据管理，若用于生产一般需要搭配第三方的如 InfulxDB 进行使用。大数据量的场景下，指标的收集和管理性能存在一定的瓶颈。


```bash
下载victoriametrics
wget https://github.com/VictoriaMetrics/VictoriaMetrics/releases/download/v1.93.16/victoria-metrics-linux-amd64-v1.93.16.tar.gz

解压软件包 
[root@prometheus-server32 ~]# tar xf victoria-metrics-linux-amd64-v1.93.16.tar.gz -C /usr/local/bin/

编写启动脚本
cat > /etc/systemd/system/victoria-metrics.service <<EOF
[Unit]
Description= Linux VictoriaMetrics Server
Documentation=https://docs.victoriametrics.com/
After=network.target

[Service]
ExecStart=/usr/local/bin/victoria-metrics-prod  \
   -httpListenAddr=0.0.0.0:8428 \
   -storageDataPath=/data/victoria-metrics \
   -retentionPeriod=6

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now victoria-metrics.service
systemctl status victoria-metrics

检查端口是否存活
[root@prometheus-server32 ~]# ss -ntl | grep 8428
LISTEN 0      4096         0.0.0.0:8428      0.0.0.0:* 

查看webUI
http://10.0.0.32:8428/
```

**prometheus配置VictoriaMetrics远端存储**

```bash
修改prometheus的配置文件
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 

...
  - job_name: node-exporters
    metrics_path: /metrics
    scheme: http
    static_configs:
    - targets:
      - 10.0.0.41:9100
      - 10.0.0.42:9100
      - 10.0.0.43:9100
    
# 在顶级字段中配置VictoriaMetrics地址
remote_write:
  - url: http://10.0.0.32:8428/api/v1/write


停止prometheus服务
[root@prometheus-server31 ~]# systemctl stop prometheus-server

手动启动prometheus服务，因为启动脚本定义了之前普罗米修斯的数据目录，这里是要将之后的数据写入到vtmetrics，所以需要手动起服务
[root@prometheus-server31 ~]# /softwares/prometheus-2.53.2.linux-amd64/prometheus    --config.file=/softwares/prometheus-2.53.2.linux-amd64/prometheus.yml
```

**vtmetrics查看数据**

```bash
node_cpu_seconds_total{instance="10.0.0.41:9100"}
```

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/85bfa65bb4094b33b86d2302c22c293f.png)



**配置grafana数据源和url**

这里数据源更改为mtmstrics的地址

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/5bbee7effec2429ca77dd12340f75053.png)


**导入模板**

```bash
1806
```

## 28、altermanager监控告警

用于prometheus server的告警功能的组件，目前支持多种告警媒介，包括但不限于邮件告警，钉钉告警，企业微信告警等。

**部署altermanager组件**

```bash
1.下载软件包
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz

2.解压软件包
[root@prometheus-server32 ~]# tar xf alertmanager-0.27.0.linux-amd64.tar.gz -C /softwares/

3 修改alermanager的配置文件
[root@prometheus-server32 ~]# cat > /softwares/alertmanager-0.27.0.linux-amd64/alertmanager.yml <<'EOF'
global:
  resolve_timeout: 5m
  smtp_from: 'y10539035@qq.com'
  smtp_smarthost: 'smtp.qq.com:465'
  smtp_auth_username: 'y10534035@qq.com'
  smtp_auth_password: 'nvkhwupusuxubefe'
  smtp_require_tls: false
  smtp_hello: 'qq.com'
route:
  group_by: ['alertname']
  group_wait: 5s
  group_interval: 5s
  repeat_interval: 5m
  receiver: 'email'
receivers:
- name: 'email'
  email_configs:
  - to: 'y10534135@qq.com'
    send_resolved: true
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
EOF

启动alermanager并访问webUI
[root@prometheus-server32 ~]# cd /softwares/alertmanager-0.27.0.linux-amd64/
[root@prometheus-server32 alertmanager-0.27.0.linux-amd64]# ./alertmanager 

--------------------------------------------------------
相关参数说明:
global:
  resolve_timeout:
  	解析超时时间。
  smtp_from:
  	发件人邮箱地址。
  smtp_smarthost:
  	邮箱的服务器的地址及端口，例如:  'smtp.qq.com:465'。
  smtp_auth_username:
  	发送人的邮箱用户名。
  smtp_auth_password:
  	发送人的邮箱授权码。
  smtp_require_tls:
  	是否基于tls加密。
  smtp_hello:
  	邮箱服务器，例如: 'qq.com'。
route:
  group_by: ['alertname']
  group_wait: 5s
  group_interval: 5s
  repeat_interval:
  	重复报警的间隔时间，如果没有解即报警问题，则会间隔指定时间一直触发报警，比如:5m。
  receiver: 
  	采用什么方式接收报警，例如'email'。
receivers:
- name: 
	定义接收者的名称，注意这里的name要和上面的route对应，例如: 'email'
  email_configs:
  - to: 
  	邮箱发给谁。
    send_resolved: true
inhibit_rules:
  - source_match:
      severity: 
      	匹配报警级别，例如: 'critical'。
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
   
--------------------------------------------------------------------------
prometheus配置alermanager作为告警媒介
1 修改配置文件
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
...
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 10.0.0.32:9093
rule_files:
  - "/softwares/prometheus-2.53.2.linux-amd64/rules/linux92.yml"


...
scrape_configs:
  ...
  - job_name: node-exporter
    static_configs:
    - targets:
      - 10.0.0.41:9100
      - 10.0.0.42:9100
      - 10.0.0.43:9100
...

2 修改告警规则
[root@prometheus-server31 ~]# mkdir -pv /softwares/prometheus-2.53.2.linux-amd64/rules

[root@prometheus-server31 ~]# cat >  /softwares/prometheus-2.53.2.linux-amd64/rules/linux.yml << EOF
groups:
- name: container-runtime
  rules:
  - alert: container-42节点挂掉啦
    expr: up{instance="10.0.0.42:9100"} == 0
    for: 15s
    labels:
      school: 001
      class: 002
    annotations:
      summary: "{{ $labels.instance }} 已停止运行超过 15s！"
  - alert: container-43节点挂掉啦
    expr: up{instance="10.0.0.43:9100"} == 0
    for: 15s
    labels:
      school: 001
      class: 002
    annotations:
      summary: "{{ $labels.instance }} 联邦模式已停止运行超过 15s！"
EOF

3 检查配置并重新加载prometheus的配置
[root@prometheus-server31 ~]# check 
[root@prometheus-server31 ~]# rr

4 查看prometheus server的WebUI
http://10.0.0.31:9090/target

5 查看alermanager的WebUI
h/softwares/alertmanager-0.27.0.linux-amd64

[root@prometheus-server32 alertmanager-0.27.0.linux-amd64]# mkdir tmpl

2 创建模板实例，工作中可以考虑嵌入公司的logo
[root@prometheus-server32 alertmanager-0.27.0.linux-amd64]# cat > tmpl/email.tmp1 <<'EOF' 
{{ define "001.html" }}
<h1 style='color: red;'>啦啦啦:  https://www.cherry.com/</h1>
<table border="1">
        <tr>
                <th>报警项</th>
                <th>实例</th>
                <th>报警阀值</th>
                <th>开始时间</th>
        </tr>
        {{ range $i, $alert := .Alerts }}
                <tr>
                        <td>{{ index $alert.Labels "alertname" }}</td>
                        <td>{{ index $alert.Labels "instance" }}</td>
                        <td>{{ index $alert.Annotations "value" }}</td>
                        <td>{{ $alert.StartsAt }}</td>
                </tr>
        {{ end }}
</table>

<img src="https://www.cherry.com/static/images/header/logo.png">

{{ end }}
EOF

3 alertmanager引用自定义模板文件
[root@prometheus-server32 alertmanager-0.27.0.linux-amd64]# cat alertmanager.yml 
global:
  resolve_timeout: 5m
  smtp_from: '31013067@qq.com'
  smtp_smarthost: 'smtp.qq.com:465'
  smtp_auth_username: '31013067@qq.com'
  smtp_auth_password: 'ysfkvbpjeddhbi'
  smtp_require_tls: false
  smtp_hello: 'qq.com'

route:
  group_by: ['alertname']
  group_wait: 5s
  group_interval: 5s
  repeat_interval: 5m
  receiver: 'email'

templates:
  - './tmp1/*.tmp1'

receivers:
- name: 'email'
  email_configs:
  - to: '31013067@qq.com'
    send_resolved: true
    headers: { Subject: "[WARN] 报警邮件" }
    html: '{{ template "cherry.html" . }}'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']

4 alertmanager语法检查
[root@prometheus-server32 alertmanager-0.27.0.linux-amd64]# pwd
/softwares/alertmanager-0.27.0.linux-amd64

[root@prometheus-server32 alertmanager-0.27.0.linux-amd64]# ./amtool check-config ./alertmanager.yml 

5 重新加载配置信息
[root@prometheus-server32 alertmanager-0.27.0.linux-amd64]# ./alertmanager 

6 查看WebUi观察配置是否生效
http://10.0.0.32:9093/#/status

---------------------------------------------
prometheus需要修改以下规则文件
1 修改规则文件
[root@prometheus-server31 ~]# cat /softwares/prometheus-2.53.2.linux-amd64/rules/linux92.yml 
groups:
- name: linux92-container-runtime
  rules:
  - alert: container-42节点挂掉啦
    expr: up{instance="10.0.0.42:9100"} == 0
    for: 15s
    labels:
      school: 001
      class: 002
    annotations:
      summary: "{{ .instance }} 已停止运行超过 15s！"
	  # 添加此行用于获取阈值
      value: "{{ $value }}"
  - alert: container-43节点的挂掉啦
    expr: up{instance="10.0.0.43:9100"} == 0
    for: 15s
    labels:
      school: 01
      class: 02
    annotations:
      summary: "{{ .instance }} 联邦模式已停止运行超过 15s！"
	  # 添加此行用于获取阈值
      value: "{{ $value }}"
      
检查语法并重新加载配置文件
[root@prometheus-server31 ~]# check 
[root@prometheus-server31 ~]# rr
```



## 29、监控K8S集群

**prometheus-operator**

prometheus-operator可以一键实现对K8S集群的监控

```bash
GitHub地址: https://github.com/prometheus-operator/kube-prometheus

基于K8S版本选择合适的prometheus-operator
https://github.com/prometheus-operator/kube-prometheus#compatibility
```



#### 1.prometheus内部监控k8s集群

普罗米修斯可以部署在k8s内部，也可以部署在k8s外部，企业中一般都是部署在k8s内部

**在K8S集群部署prometheus**

```bash
下载软件包
wget https://github.com/prometheus-operator/kube-prometheus/archive/refs/tags/v0.11.0.tar.gz

解压软件包
[root@master231 ~]# tar xf kube-prometheus-0.11.0.tar.gz -C /softwares/

切换工作目录，进入到prometheus-operator主目录
[root@master231 ~]# cd /softwares/kube-prometheus-0.11.0/

更改yaml文件，自定义资源
[root@master231 kube-prometheus-0.11.0]# vim manifests/prometheusAdapter-deployment.yaml
......
        # image: k8s.gcr.io/prometheus-adapter/prometheus-adapter:v0.9.1
        image: registry.cn-hangzhou.aliyuncs.com/k8s/prometheus-adapter:v0.9.1
...

[root@master231 kube-prometheus-0.11.0]# vim manifests/kubeStateMetrics-deployment.yaml
...
        # image: k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.5.0
        image: registry.cn-hangzhou.aliyuncs.com/k8s/kube-state-metrics:2.5.0
        
[root@master231 kube-prometheus-0.11.0]# vim manifests/grafana-service.yaml
....
spec:
  ...
  type: NodePort
  ports:
  - name: http
    port: 3000
    targetPort: http
    nodePort: 30080
    
[root@master231 kube-prometheus-0.11.0]# vim manifests/prometheus-service.yaml
....
spec:
  type: NodePort
  ports:
  - name: web
    port: 9090
    targetPort: web
    nodePort: 30090
    
部署服务
[root@master231 kube-prometheus-0.11.0]# kubectl apply --server-side -f manifests/setup
[root@master231 kube-prometheus-0.11.0]# kubectl wait \
	--for condition=Established \
	--all CustomResourceDefinition \
	--namespace=monitoring
[root@master231 kube-prometheus-0.11.0]# kubectl apply -f manifests/

查看对应的Pod运行列表 ------>如果没运行起来，一般手动导入镜像到对应节点
[root@master231 kube-prometheus-0.11.0]# kubectl get pods -n monitoring  -o wide
删除镜像，重新拉取
[root@master231 kube-prometheus-0.11.0]# kubectl -n monitoring delete pod --all
查看pod事件信息
[root@master231 kube-prometheus-0.11.0]# kubectl -n monitoring describe prometheus-k8s-0
查看所有服务
[root@master231 kube-prometheus-0.11.0]# kubectl get svc -A
查看svc详情
[root@master231 kube-prometheus-0.11.0]# kubectl describe svc -n monitoring prometheus-k8s

修改收件人和发件人信息
[root@master231 kube-prometheus-0.11.0]# vim manifests/alertmanager-secret.yaml 
	里面记录了alertmanager的收件人和发件人信息。
	
访问WebUI 
grafana：账号密码admin
http://10.0.0.231:30080
------------------------------
普罗米修斯：
http://10.0.0.231:30090
------------------------------

查看内置的模板 
查看后再倒入1860模板对比测试。
```



#### 2.prometheus外部监控k8s集群

##### **监控node-exporter节点**

```bash
1.所有节点导入镜像
[root@master231 ~]# docker load -i node-exporter_v1.8.1.tar.gz

2.在k8smaster编写资源清单
[root@master231 ~]# cat ds-node-exporter.yaml 
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ds-node-exporter
spec:
  selector:
    matchLabels:
      apps: node-exporter
  template:
    metadata:
      labels:
        apps: node-exporter
    spec:
      hostNetwork: true
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.8.1
        command:
        - /bin/node_exporter
        - --web.listen-address=:19100

3.查看pod ---> 每个节点成功运行
[root@master231 ~]# kubectl get pods -o wide
NAME                     READY   STATUS        RESTARTS   AGE    IP           NODE      
ds-node-exporter-5b4gc   1/1     Running       0          35s    10.0.0.233   worker233 
ds-node-exporter-dmnnj   1/1     Running       0          35s    10.0.0.232   worker232 
ds-node-exporter-hpj9h   1/1     Running       0          35s    10.0.0.231   master231 

4.修改Prometheus的配置文件并重新加载
[root@prometheus-server31 ~]# vim /softwares/prometheus-2.53.2.linux-amd64/prometheus.yml 
...
  - job_name: k8s-node-exporter
    static_configs:
    - targets:
      - 10.0.0.231:19100
      - 10.0.0.232:19100
      - 10.0.0.233:19100
      
4.访问Prometheus的WebUI
http://10.0.0.31:9090/targets

5.grafana采集普罗米修斯31数据源的信息，导入模板ID
1860

```

##### **监控云原生应用etcd案例**

```bash
1.查看etcd证书存储路径
[root@master231 ~]#  egrep "\--key-file|--cert-file" /etc/kubernetes/manifests/etcd.yaml 
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --key-file=/etc/kubernetes/pki/etcd/server.key

2 测试etcd证书访问的metrics接口
[root@master231 ~]# curl -s --cert /etc/kubernetes/pki/etcd/server.crt  --key /etc/kubernetes/pki/etcd/server.key https://10.0.0.231:2379/metrics -k | tail

3. 创建etcd的service
[root@master231 ~]# cat etcd-svc.yaml 
apiVersion: v1
kind: Endpoints
metadata:
  name: etcd-k8s
  namespace:  kube-system
subsets:
- addresses:
  - ip: 10.0.0.231
  ports:
  - name: https-metrics
    port: 2379
    protocol: TCP

---

apiVersion: v1
kind: Service
metadata:
  name: etcd-k8s
  namespace: kube-system
  labels:
    apps: etcd
spec:
  ports:
  - name: https-metrics
    port: 2379
    targetPort: 2379
  type: ClusterIP
  
[root@master231 ~]# kubectl apply -f etcd-svc.yaml
[root@master231 ~]# kubectl get svc -n kube-system -l apps=etcd
NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
etcd-k8s   ClusterIP   10.200.33.157   <none>        2379/TCP   36m
[root@master231 ~]# 
[root@master231 ~]# kubectl -n kube-system describe svc etcd-k8s  | grep Endpoints
Endpoints:         10.0.0.231:2379


3.基于创建的svc访问测试连通性
[root@master231 ~]# curl -s --cert /etc/kubernetes/pki/etcd/server.crt  --key /etc/kubernetes/pki/etcd/server.key https://10.200.33.157:2379/metrics -k | tail -1 
promhttp_metric_handler_requests_total{code="503"} 0

4.创建etcd证书的secrets并挂载到Prometheus server
		4.1 查找需要挂载etcd的证书文件路径
[root@master231 ~]# egrep "\--key-file|--cert-file|--trusted-ca-file" /etc/kubernetes/manifests/etcd.yaml   
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
[root@master231 ~]# 
	
	
		4.2 根据etcd的实际存储路径创建secrets
[root@master231 ~]# kubectl create secret generic etcd-tls --from-file=/etc/kubernetes/pki/etcd/server.crt --from-file=/etc/kubernetes/pki/etcd/server.key  --from-file=/etc/kubernetes/pki/etcd/ca.crt -n monitoring 
secret/etcd-tls created
[root@master231 ~]# 
[root@master231 ~]# kubectl -n monitoring get secrets etcd-tls 
NAME       TYPE     DATA   AGE
etcd-tls   Opaque   3      12s
[root@master231 ~]# 


		4.3 修改Prometheus的资源，修改后会自动重启
[root@master231 ~]# kubectl -n monitoring edit prometheus k8s
...
spec:
  secrets:
  - etcd-tls
  ...  
[root@master231 ~]# kubectl -n monitoring get pods -l app.kubernetes.io/component=prometheus -o wide
NAME               READY   STATUS    RESTARTS   AGE   IP            NODE        NOMINATED NODE   READINESS GATES
prometheus-k8s-0   2/2     Running   0          74s   10.100.1.57   worker232   <none>           <none>
prometheus-k8s-1   2/2     Running   0          92s   10.100.2.28   worker233   <none>           <none>
[root@master231 ~]# 


		4.4 查看证书是否挂载成功
[root@master231 ~]# kubectl -n monitoring exec prometheus-k8s-0 -c prometheus -- ls -l /etc/prometheus/secrets/etcd-tls
total 0
lrwxrwxrwx    1 root     2000            13 Jan 24 14:07 ca.crt -> ..data/ca.crt
lrwxrwxrwx    1 root     2000            17 Jan 24 14:07 server.crt -> ..data/server.crt
lrwxrwxrwx    1 root     2000            17 Jan 24 14:07 server.key -> ..data/server.key
[root@master231 ~]# 
[root@master231 ~]# kubectl -n monitoring exec prometheus-k8s-1 -c prometheus -- ls -l /etc/prometheus/secrets/etcd-tls
total 0
lrwxrwxrwx    1 root     2000            13 Jan 24 14:07 ca.crt -> ..data/ca.crt
lrwxrwxrwx    1 root     2000            17 Jan 24 14:07 server.crt -> ..data/server.crt
lrwxrwxrwx    1 root     2000            17 Jan 24 14:07 server.key -> ..data/server.key
[root@master231 ~]# 


5.创建ServerMonitor
		5.1 创建ServiceMonitor资源关联etcd的svc
[root@master231 ~]# cat  etcd-smon.yaml 
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: etcd-smon
  namespace: monitoring
spec:
  # 指定job的标签，可以不设置。
  jobLabel: kubeadm-etcd-k8s
  # 指定监控后端目标的策略
  endpoints:
    # 监控数据抓取的时间间隔
  - interval: 30s
    # 指定metrics端口，这个port对应Services.spec.ports.name
    port: https-metrics
    # Metrics接口路径
    path: /metrics
    # Metrics接口的协议
    scheme: https
    # 指定用于连接etcd的证书文件
    tlsConfig:
      # 指定etcd的CA的证书文件
      caFile:  /etc/prometheus/secrets/etcd-tls/ca.crt
      # 指定etcd的证书文件
      certFile: /etc/prometheus/secrets/etcd-tls/server.crt
      # 指定etcd的私钥文件
      keyFile: /etc/prometheus/secrets/etcd-tls/server.key
      # 关闭证书校验，毕竟咱们是自建的证书，而非官方授权的证书文件。
      insecureSkipVerify: true
  # 监控目标Service所在的命名空间
  namespaceSelector:
    matchNames:
    - kube-system
  # 监控目标Service目标的标签。
  selector:
    # 注意，这个标签要和etcd的service的标签保持一致哟
    matchLabels:
      apps: etcd
[root@master231 ~]# 
[root@master231 ~]# kubectl apply -f etcd-smon.yaml 
servicemonitor.monitoring.coreos.com/etcd-smon created
[root@master231 ~]# 
[root@master231 ~]# kubectl get smon -n monitoring yinzhengjie-etcd-smon 
NAME                    AGE
yinzhengjie-etcd-smon   8s
[root@master231 ~]# 


5.2.访问Prometheus的WebUI
http://10.0.0.233:30090/targets?search=

	
6.查看etcd的数据
etcd_cluster_version

7.使用grafana查看etcd数据
http://10.0.0.233:30080/?orgId=1
3070
```