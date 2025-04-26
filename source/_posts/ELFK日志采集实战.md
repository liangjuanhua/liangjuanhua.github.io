---
title: ELFK日志采集实战
date: 2025-04-16 16:18:30
tags: 日志篇  
categories: 日志篇
---
#  一、日志分析概述

- 日志分析是运维工程师解决系统故障，发现问题的主要手段
- 日志主要包括系统日志、应用程序日志和安全日志
- 系统运维和开发人员可以通过日志了解服务器软硬件信息、检查配置过程中的错误及错误发生的原因
- 经常分析日志可以了解服务器的负荷，性能安全性，从而及时采取措施纠正错误

# 二、日志分析的作用

- 分析日志时刻监控系统运行的状态
- 分析日志来定位程序的bug
- 分析日志监控网站的访问流量
- 分析日志可以知道哪些sql语句需要优化

# 三、ELK概述

ELFK 已经成为目前最流行的集中式日志解决方案，它主要是由 filebeat、Logstash、Elasticsearch、Kibana 等组件组成，来共同完成实时日志的收集，存储，展示等一站式的解决方案。本文将会介绍 ELK 常见的架构以及相关问题解决。

- Filebeat：Filebeat 是一款轻量级，占用服务资源非常少的数据收集引擎，它是 ELK 家族的新成员，可以代替 Logstash 作为在应用服务器端的日志收集引擎，支持将收集到的数据输出到 Kafka，Redis 等队列。
- Logstash：数据收集引擎，相较于 Filebeat 比较重量级，但它集成了大量的插件，支持丰富的数据源收集，对收集的数据可以过滤，分析，格式化日志格式。
- Elasticsearch：分布式数据搜索引擎，基于 Apache Lucene 实现，可集群，提供数据的集中式存储，分析，以及强大的数据搜索和聚合功能。
- Kibana：数据的可视化平台，通过该 web 平台可以实时的查看 Elasticsearch 中的相关数据，并提供了丰富的图表统计功能

# 四、日志采集常见部署架构

**filebeat作为日志收集器**

使用 Filebeat 作为日志采集器，通常可以将其直接作为日志收集的前端组件，负责读取日志文件并发送到后续Elasticsearch 中。之后kabana做出图展示。

![2b05ddcb77964793a6825be117a9af44.png](https://gitee.com/ljh00928/csdn/raw/master/img/2b05ddcb77964793a6825be117a9af44.png)

**引入缓存队列的部署架构**

该架构在第二种架构的基础上引入了 Kafka 消息队列（还可以是其他消息队列），将 Filebeat 收集到的数据发送至 Kafka，然后在通过 Logstasth 读取 Kafka 中的数据，这种架构主要是解决大数据量下的日志收集方案，使用缓存队列主要是解决数据安全与均衡 Logstash 与 Elasticsearch 负载压力。

![d0372a55a98241e1a77a402e5a7bd5d5.png](https://gitee.com/ljh00928/csdn/raw/master/img/d0372a55a98241e1a77a402e5a7bd5d5.png)



> 本次我们部署efk架构。filebeat作为日志采集器，Elasticsearch存储数据，kibana展示采集的数据。

### **部署 Elasticsearch**

```
1.下载软件包
[root@node1.local ~]# wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.22-amd64.deb

2.安装
[root@node1.local ~]# dpkg -i elasticsearch-7.17.22-amd64.deb

3.修改配置文件
[root@node1.local ~]# vim /etc/elasticsearch/elasticsearch.yml 
cluster.name: cherry
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300
discovery.type: "single-node"

4.启动服务
[root@node1.local ~]# systemctl enable --now elasticsearch.service
Synchronizing state of elasticsearch.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable elasticsearch
Created symlink /etc/systemd/system/multi-user.target.wants/elasticsearch.service → /usr/lib/systemd/system/elasticsearch.service.

5.查看端口验证服务是否启动成功
[root@node1.local ~]# ss -ntl | grep "9[2|3]00"
LISTEN 0      65535              *:9300             *:*          
LISTEN 0      65535              *:9200             *:*  

6.访问
[root@node1.local ~]# curl http://192.1.7.244:9200/
{
  "name" : "node1.local",
  "cluster_name" : "cherry",
  "cluster_uuid" : "1-AfkL1DRXKIMuhaQuoFLQ",
  "version" : {
    "number" : "7.17.22",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "38e9ca2e81304a821c50862dafab089ca863944b",
    "build_date" : "2024-06-06T07:35:17.876121680Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.3",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}

7.查看节点信息
[root@node1.local ~]# curl 192.1.7.244:9200/_cat/nodes
192.1.7.244 18 99 2 0.05 0.15 0.07 cdfhilmrstw * node1.local
[root@node1.local ~]#
[root@node1.local ~]# curl 192.1.7.244:9200/_cat/nodes?v
ip          heap.percent ram.percent cpu load_1m load_5m load_15m node.role   master name
192.1.7.244           18          99   0    0.04    0.14     0.07 cdfhilmrstw *      node1.local
```

### 部署kibana

```
1.下载kibana
[root@node1.local ~]# wget https://artifacts.elastic.co/downloads/kibana/kibana-7.17.22-amd64.deb

2.安装
[root@node1.local ~]# dpkg -i kibana-7.17.22-amd64.deb

3.修改配置文件
[root@node1.local ~]# vim /etc/kibana/kibana.yml
...
# 监听的端口
server.port: 5601
# 监听的地址
server.host: "0.0.0.0"
# 指定ES集群地址
elasticsearch.hosts: ["http://192.1.7.244:9200"]
# 国际化语言
i18n.locale: "zh-CN"

4.启动
[root@node1.local ~]# systemctl enable --now kibana.service
[root@node1.local ~]# ss -ntl | grep 5601
LISTEN 0      511          0.0.0.0:5601      0.0.0.0:* 

5.访问
[root@node1.local ~]# curl -I http://192.1.7.244:5601
HTTP/1.1 302 Found
location: /spaces/enter
x-content-type-options: nosniff
referrer-policy: no-referrer-when-downgrade
content-security-policy: script-src 'unsafe-eval' 'self'; worker-src blob: 'self'; style-src 'unsafe-inline' 'self'
kbn-name: node1.local
kbn-license-sig: f4384e4807b75451c2f4feacc4d54e13f9397d4b4f115c922f956f9017453f91
cache-control: private, no-cache, no-store, must-revalidate
content-length: 0
Date: Thu, 09 Jan 2025 05:51:05 GMT
Connection: keep-alive
Keep-Alive: timeout=120
```

### 部署filebeat

![](https://gitee.com/ljh00928/csdn/raw/master/img/5ade8e793739408487a70a3943757ab7.png)

```
1.下载
[root@node1.local ~]# wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.17.22-amd64.deb

2.安装
[root@node1.local ~]# dpkg -i filebeat-7.17.22-amd64.deb

3.验证是否安装成功
[root@node1.local ~]# filebeat -h
```

**filebeat组成部分**

> input:数据从哪里来？
>  output：数据去哪里？

**filebeat结构**

> \- input: 数据从哪来？
>    \- stdin
>    \- tcp
>    \- log
>
> 
>  \- output: 数据到哪去？
>    \- console
>    \- elasticsearch

# 五、测试

定义filebeat采集的日志索引模板

```
[root@node1.local filebeat]# vim systemlog.yaml 
filebeat.inputs:
- type: log
  paths:
    - /var/log/*.log

output.elasticsearch:
  hosts:
  - "http://192.1.7.244:9200"
  index: "system-log"

# 禁用索引的生命周期(Index Lifecycle Management,简称"ilm")，如果不禁用则忽略自定义索引名称
setup.ilm.enabled: false
# ES的索引模板名称
setup.template.name: "system"
# ES索引模板的匹配模式
setup.template.pattern: "system-log*"
# 如果索引模板已经存在，是否覆盖，推荐设置为false
setup.template.overwrite: false
# 设置索引模板
setup.template.settings:
  # 设置索引的分片数量
  index.number_of_shards: 5
  # 设置索引的副本数量
  index.number_of_replicas: 0
```

**执行filebeat采集日志**

```
[root@node1.local filebeat]# filebeat -e -c /root/filebeat/systemlog.yaml 
```

**查看索引** 

![89899c03a9b041cfaf7b981f14cbe78a.png](https://gitee.com/ljh00928/csdn/raw/master/img/89899c03a9b041cfaf7b981f14cbe78a.png)

**添加索引** 

![b5b5a1444258447baeedc342957cc623.png](https://gitee.com/ljh00928/csdn/raw/master/img/b5b5a1444258447baeedc342957cc623.png)

**查看日志** 

![3e9aa0ea8f8e41a7b0154f7afe40e9ef.png](https://gitee.com/ljh00928/csdn/raw/master/img/3e9aa0ea8f8e41a7b0154f7afe40e9ef.png)

# 六、logstash

filebeat是基于logstash再次开发的，logstash可以对自己公司自研产品进行自定义日志，选择要截取的日志。

Logstash 是一个具有实时管道功能的开源数据收集引擎。Logstash可以动态统一来自不同来源的数据，并将数据规范化到您选择的目标中。为了多样化的高级下游分析和可视化用例，清理和使所有数据平等化。

虽然 Logstash 最初在日志收集方面推动了创新，但它的能力远远超出了该用例。任何类型的事件都可以通过广泛的输入、过滤和输出插件进行增强和转换，许多本地编解码器进一步简化了摄入过程。Logstash 通过利用更多的数据量和种类加速您的洞察力。

**工作原理**

Logstash 事件处理管道有三个阶段：**输入 → 过滤器 → 输出**。

inputs 模块负责收集数据，filters 模块可以对收集到的数据进行格式化、过滤、简单的数据处理，outputs 模块负责将数据同步到目的地，Logstash的处理流程，就像管道一样，数据从管道的一端，流向另外一端。

inputs 和 outputs 支持编解码器，使您能够在数据进入或离开管道时对数据进行编码或解码，而无需使用单独的过滤器。

![27f6c2a9e55249deb6c91357a5cccb3e.png](https://gitee.com/ljh00928/csdn/raw/master/img/27f6c2a9e55249deb6c91357a5cccb3e.png)

**部署logstash**

```
1.下载logstash
[root@node1.local ~]#  wget https://artifacts.elastic.co/downloads/logstash/logstash-7.17.22-amd64.deb

2.安装logstash
[root@node1.local ~]#  dpkg -i logstash-7.17.22-amd64.deb

3.配置环境变量
[root@node1.local ~]#  ln -svf /usr/share/logstash/bin/logstash /usr/local/sbin/
'/usr/local/sbin/logstash' -> '/usr/share/logstash/bin/logstash'
```

**logstash结构**

> \- pipeline
>    \- input
>    \- stdin
>    \- file 
>  \- filter
>    \- mutate
>  \- output
>    \- stdout
>    \- elasticsearch

**fileter插件**

> grok：logstash 中最常用的日志解释和结构化插件。
>
> grok ：是一种采用组合多个预定义的正则表达式，用来匹配分割文本并映射到关键字的工具。
>  mutate ：支持事件的变换，例如重命名、移除、替换、修改等
>
> drop ：完全丢弃事件
>
> clone ：克隆事件
>
> geoip：添加关于 IP 地址的地理位置信息
>
> date：专门用于处理日志的日期模块，将其作为时间戳可以替换"@timestamp"字段。

**案例编写**

```
[root@node1.local conf.d]# vim 03-file-filter-elasticsearch.conf 
input {
  # 输入类型是一个file，代表的是文本文件
  file {
    # 指定的文件的路径，可以指定多个
    path => ["/var/log/*.log"]
    # 指定从源文件采集的位置，有效值为: beginning, end(默认值)。
    # 该参数仅在首次采集(没有在".sincedb*"文件中记录)新文件时生效，第二次采集则无视此参数。
    start_position => "beginning"
  }
}

filter {
  mutate {
      # 对指定字段进行切分，本案例是将message字段按照"|"进行切分
      split => { "message" => "|" }

      # 添加字段
      add_field => { 
         "other" => "%{[message][0]}"
         "userId" => "%{[message][1]}" 
         "action" => "%{[message][2]}"
         "svip" => "%{[message][3]}"
         "price" => "%{[message][4]}" 

      }
  }

  mutate {
      split => { "other" => " "}
      add_field => {
         datetime => "%{[other][1]} %{[other][2]}"
      }

      # 移除字段
      remove_field => [ "message" , "other", "@version"]
  }


  mutate {
     # 对字段进行数据转换
     convert => {
       "price" => "float"
       "userId" => "integer"
     }
  }


   # 处理时间相关的模块
   date {
     # 匹配日期字段，将"datetime" 转换为日期格式
     #    源数据： "datetime" => "2024-07-16 10:34:39"
     match => [ "datetime", "yyyy-MM-dd HH:mm:ss" ]

     # 使用match匹配到的时间数据类型存储在哪个字段中，若不指定，则默认使用覆盖"@timestamp"
     # target => "datetime"
   }


}

output {
  stdout {}

 elasticsearch {
    # 指定ES集群地址
    hosts => ["http://192.1.7.244:9200"]
    # 指定ES自定义索引的名称
    index => "system-%{+yyyy.MM.dd}"
 }
}
```

 **启动logstash实例**

```
[root@node1.local ~]#  logstash -rf /etc/logstash/conf.d/03-file-filter-elasticsearch.conf
```

![5bcd72e718c142c0a1cd2f9eca041d40.png](https://gitee.com/ljh00928/csdn/raw/master/img/5bcd72e718c142c0a1cd2f9eca041d40.png)

**kibana查看索引**

![da51a954f4b244cabd5180a97cedf398.png](https://gitee.com/ljh00928/csdn/raw/master/img/da51a954f4b244cabd5180a97cedf398.png)

**创建索引**

![8fdb73bd47414486b46efcb9f324d2cd.png](https://gitee.com/ljh00928/csdn/raw/master/img/8fdb73bd47414486b46efcb9f324d2cd.png)

**查看日志数据**

很明显，相对于filebeat来说。logstash这时候已经精准过滤出我们需要的字段了

![1b492473f66f4d6cbe1512f88d24c6ff.png](https://gitee.com/ljh00928/csdn/raw/master/img/1b492473f66f4d6cbe1512f88d24c6ff.png)

# 七、常见问题及解决方案

**问题：如何实现日志的多行合并功能？**

系统应用中的日志一般都是以特定格式进行打印的，属于同一条日志的数据可能分多行进行打印，那么在使用 ELK 收集日志的时候就需要将属于同一条日志的多行数据进行合并。

解决方案：使用 Filebeat 或 Logstash 中的 multiline 多行合并插件来实现

在使用 multiline 多行合并插件的时候需要注意，不同的 ELK 部署架构可能 multiline 的使用方式也不同，如果是本文的第一种部署架构，那么 multiline 需要在 Logstash 中配置使用，如果是第二种部署架构，那么 multiline 需要在 Filebeat 中配置使用，无需再在 Logstash 中配置 multiline。

**1、multiline 在 Filebeat 中的配置方式：**

![c5e5bf831c80439592004b44b09a6836.png](https://gitee.com/ljh00928/csdn/raw/master/img/c5e5bf831c80439592004b44b09a6836.png)

> - negate：默认为 false，表示匹配 pattern 的行合并到上一行；true 表示不匹配 pattern 的行合并到上一行
> - match：after 表示合并到上一行的末尾，before 表示合并到上一行的行首

如：该配置表示将不匹配 pattern 模式的行合并到上一行的末尾

```
pattern: '\['
negate: true
match: after
```

**2、multiline 在 Logstash 中的配置方式**

![c0f265a20f22429688c3db433c48de05.png](https://gitee.com/ljh00928/csdn/raw/master/img/c0f265a20f22429688c3db433c48de05.png)

> （1）Logstash 中配置的 what 属性值为 previous，相当于 Filebeat 中的 after，Logstash 中配置的 what 属性值为 next，相当于 Filebeat 中的 before。
>  （2）pattern => "%{LOGLEVEL}\s*\]" 中的 LOGLEVEL 是 Logstash 预制的正则匹配模式，预制的还有好多常用的正则匹配模式，详细请看：https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns 

**问题：如何将 Kibana 中显示日志的时间字段替换为日志信息中的时间？** 

默认情况下，我们在 Kibana 中查看的时间字段与日志信息中的时间不一致，因为默认的时间字段值是日志收集时的当前时间，所以需要将该字段的时间替换为日志信息中的时间。

**解决方案：使用 grok 分词插件与 date 时间格式化插件来实现**

在 Logstash 的配置文件的过滤器中配置 grok 分词插件与 date 时间格式化插件，如：

![37091060f3d441b384738b4655ec0297.png](https://gitee.com/ljh00928/csdn/raw/master/img/37091060f3d441b384738b4655ec0297.png)

如要匹配的日志格式为：“[DEBUG][20170811 10:07:31,359][DefaultBeanDefinitionDocumentReader:106] Loading bean definitions”，解析出该日志的时间字段的方式有：

① 通过引入写好的表达式文件，如表达式文件为 customer_patterns，内容为：
 CUSTOMER_TIME %{YEAR}%{MONTHNUM}%{MONTHDAY}\s+%{TIME}
 注：内容格式为：[自定义表达式名称] [正则表达式]
 然后 logstash 中就可以这样引用：

![f4a603385f984c268b4171c088fa516b.png](https://gitee.com/ljh00928/csdn/raw/master/img/f4a603385f984c268b4171c088fa516b.png)

 ② 以配置项的方式，规则为：(?< 自定义表达式名称> 正则匹配规则)，如：

```
filter {  
  grok {    
    match => [ "message" , "(?<customer_time>%{YEAR}%{MONTHNUM}%{MONTHDAY}\s+%{TIME})" ]  
  }
}
```

**问题：如何在 Kibana 中通过选择不同的系统日志模块来查看数据**

 一般在 Kibana 中显示的日志数据混合了来自不同系统模块的数据，那么如何来选择或者过滤只查看指定的系统模块的日志数据？

**解决方案：新增标识不同系统模块的字段或根据不同系统模块建 ES 索引**

1、新增标识不同系统模块的字段，然后在 Kibana 中可以根据该字段来过滤查询不同模块的数据

这里以第二种部署架构讲解，在 Filebeat 中的配置内容为：

![ddf6958c25a4490795f9de0d0e29b836.png](https://gitee.com/ljh00928/csdn/raw/master/img/ddf6958c25a4490795f9de0d0e29b836.png)

> 通过新增：log_from 字段来标识不同的系统模块日志 

2、根据不同的系统模块配置对应的 ES 索引，然后在 Kibana 中创建对应的索引模式匹配，即可在页面通过索引模式下拉框选择不同的系统模块数据。
 这里以第二种部署架构讲解，分为两步：

① 在 Filebeat 中的配置内容为：

![f0aa2ef6423d4e0887d40b3fb593c541.png](https://gitee.com/ljh00928/csdn/raw/master/img/f0aa2ef6423d4e0887d40b3fb593c541.png)

> 通过 document_type 来标识不同系统模块

 ② 修改 Logstash 中 output 的配置内容为：

![ddcb22d3baf34114b02bb12476ddf1c5.png](https://gitee.com/ljh00928/csdn/raw/master/img/ddcb22d3baf34114b02bb12476ddf1c5.png)

> 在 output 中增加 index 属性，%{type} 表示按不同的 document_type 值建 ES 索引