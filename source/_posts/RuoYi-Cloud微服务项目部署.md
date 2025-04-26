---
title: RuoYi-Cloud微服务项目部署
date: 2025-04-18 11:34:37
tags: k8s
categories: 
  - 云原生
---
# 前言

想来第一次与微服务结缘是在我的第一个 Springboot项目，我接触到了 Nacos 和 Gateway，从那时起，便对微服务产生了浓厚的兴趣，微服务的架构设计、服务拆分、服务治理等内容确实很有意思。这也是我们从事运维的原因之一。自己工作一段时间后，我想着是时候输出一些内容了，把微服务以更加通俗易懂的部署起来。这里先带大家简单的单机部署简单了解下，实际工作中，面临网关，注册中心，调度中心，数据同步选型等。其实部署到k8s中也是一样的，只是将服务容器化，使用k8s编排。如果大家感兴趣，点赞关注就是对我最大的支持。后期我会写一篇关于k8s部署Spring Cloud并结合jenkins，gitlab，harbor等做cicd流水线集成。

# 一、框架介绍

RuoYi-Cloud是一款基于Spring Boot、Spring Cloud & Alibaba、Vue、Element的前后端分离微服务极速后台开发框架。内置模块如：部门管理、角色用户、菜单及按钮授权、数据权限、系统参数、日志管理、代码生成等。在线定时任务配置；支持集群，支持多数据源。

RuoYi-Cloud 是通过Java开发，基于微服务架构设计的权限管理系统。

文档地址：http://doc.ruoyi.vip/ruoyi-cloud/

代码下载：https://gitee.com/y_project/RuoYi-Cloud 

```bash
Java程序开发框架介绍：
● Spring：是一个开源的Java开发框架，用于简化Java程序的开发。 
● Spring Boot：是基于Spring的全新框架，对比于Spring框架更加的简洁高效。
● Spring Cloud：是一套微服务开发框架。
● Spring Cloud Alibaba：是阿里巴巴推出的微服务开发框架。
```

## 架构图

```bash
架构图RuoYi-Cloud-Processon (opens new window)分享地址。
https://www.processon.com/view/5ec290195653bb6f2a18504e
```



![img](https://gitee.com/ljh00928/csdn/raw/master/img/dacd9b75d911fe2aed60fe2f0fcd8bdd.png)

##  系统模块

```bash
com.ruoyi     
├── ruoyi-ui              // 前端框架 [80]
├── ruoyi-gateway         // 网关模块 [8080]
├── ruoyi-auth            // 认证中心 [9200]
├── ruoyi-api             // 接口模块
│       └── ruoyi-api-system                          // 系统接口
├── ruoyi-common          // 通用模块
│       └── ruoyi-common-core                         // 核心模块
│       └── ruoyi-common-datascope                    // 权限范围
│       └── ruoyi-common-datasource                   // 多数据源
│       └── ruoyi-common-log                          // 日志记录
│       └── ruoyi-common-redis                        // 缓存服务
│       └── ruoyi-common-seata                        // 分布式事务
│       └── ruoyi-common-security                     // 安全模块
│       └── ruoyi-common-sensitive                    // 数据脱敏
│       └── ruoyi-common-swagger                      // 系统接口
├── ruoyi-modules         // 业务模块
│       └── ruoyi-system                              // 系统模块 [9201]
│       └── ruoyi-gen                                 // 代码生成 [9202]
│       └── ruoyi-job                                 // 定时任务 [9203]
│       └── ruoyi-file                                // 文件服务 [9300]
├── ruoyi-visual          // 图形化管理模块
│       └── ruoyi-visual-monitor                      // 监控中心 [9100]
├──pom.xml                // 公共依赖
```

## 主要特性

- 完全响应式布局（支持电脑、平板、手机等所有主流设备）
- 强大的一键生成功能（包括控制器、模型、视图、菜单等）
- 支持多数据源，简单配置即可实现切换。
- 支持按钮及数据权限，可自定义部门数据权限。
- 对常用js插件进行二次封装，使js代码变得简洁，更加易维护
- 完善的XSS防范及脚本过滤，彻底杜绝XSS攻击
- Maven多项目依赖，模块及插件分项目，尽量松耦合，方便模块升级、增减模块。
- 国际化支持，服务端及客户端支持
- 完善的日志记录体系简单注解即可实现

## 技术选型

**1、系统环境**

- Java EE 8
- Servlet 3.0
- Apache Maven 3

**2、主框架**

- Spring Boot 2.3.x
- Spring Cloud Hoxton.SR9
- Spring Framework 5.2.x
- Spring Security 5.2.x

**3、持久层**

- Apache MyBatis 3.5.x
- Hibernate Validation 6.0.x
- Alibaba Druid 1.2.x

**4、视图层**

- Vue 2.6.x
- Axios 0.21.0
- Element 2.14.x

## 内置功能

- 用户管理：用户是系统操作者，该功能主要完成系统用户配置。
- 部门管理：配置系统组织机构（公司、部门、小组），树结构展现支持数据权限。
- 岗位管理：配置系统用户所属担任职务。
- 菜单管理：配置系统菜单，操作权限，按钮权限标识等。
- 角色管理：角色菜单权限分配、设置角色按机构进行数据范围权限划分。
- 字典管理：对系统中经常使用的一些较为固定的数据进行维护。
- 参数管理：对系统动态配置常用参数。
- 通知公告：系统通知公告信息发布维护。
- 操作日志：系统正常操作日志记录和查询；系统异常信息日志记录和查询。
- 登录日志：系统登录日志记录查询包含登录异常。
- 在线用户：当前系统中活跃用户状态监控。
- 定时任务：在线（添加、修改、删除)任务调度包含执行结果日志。
- 代码生成：前后端代码的生成（java、html、xml、sql)支持CRUD下载 。
- 系统接口：根据业务代码自动生成相关的api接口文档。
- 服务监控：监视当前系统CPU、内存、磁盘、堆栈等相关信息。
- 在线构建器：拖动表单元素生成相应的HTML代码。
- 连接池监视：监视当期系统数据库连接池状态，可进行分析SQL找出系统性能瓶颈。

# 二、准备代码

gitee地址：https://gitee.com/y_project/RuoYi-Cloud

下载代码
![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/3fba7c3cb6034bc18cc2fdae4b3ce194.png)

创建代码存放目录

```bash
[root@node1.local ~]# mkdir RuoYi-Cloud

[root@node1.local ~]# cd RuoYi-Cloud/

解压zip包
[root@node1.local RuoYi-Cloud]# unzip RuoYi-Cloud-master.zip
```

查看

```bash
[root@node1.local RuoYi-Cloud]# cd RuoYi-Cloud-master/
[root@node1.local RuoYi-Cloud-master]# ll
total 88
drwxr-xr-x 13 root root  4096 Jan  7 10:55 ./
drwxr-xr-x  3 root root  4096 Feb 10 14:12 ../
drwxr-xr-x  2 root root  4096 Jan  7 10:55 bin/
drwxr-xr-x  7 root root  4096 Jan  7 10:55 docker/
drwxr-xr-x  2 root root  4096 Jan  7 10:55 .github/
-rw-r--r--  1 root root   698 Jan  7 10:55 .gitignore
-rw-r--r--  1 root root  1063 Jan  7 10:55 LICENSE
-rw-r--r--  1 root root 12420 Jan  7 10:55 pom.xml
-rw-r--r--  1 root root  9220 Jan  7 10:55 README.md
drwxr-xr-x  3 root root  4096 Jan  7 10:55 ruoyi-api/
drwxr-xr-x  3 root root  4096 Jan  7 10:55 ruoyi-auth/
drwxr-xr-x 11 root root  4096 Jan  7 10:55 ruoyi-common/
drwxr-xr-x  3 root root  4096 Jan  7 10:55 ruoyi-gateway/
drwxr-xr-x  6 root root  4096 Jan  7 10:55 ruoyi-modules/
drwxr-xr-x  6 root root  4096 Jan  7 10:55 ruoyi-ui/
drwxr-xr-x  3 root root  4096 Jan  7 10:55 ruoyi-visual/
drwxr-xr-x  2 root root  4096 Jan  7 10:55 sql/
```



# 三、环境部署

## 准备工作

```bash
JDK >= 1.8 (推荐1.8版本)
Mysql >= 5.7.0 (推荐5.7版本)
Redis >= 3.0
Maven >= 3.0
Node >= 12
nacos >= 2.0.4 (ruoyi-cloud < 3.0 需要下载nacos >= 1.4.x版本)
sentinel >= 1.6.0
```

查看jdk对应maven版本链接：http://maven.apache.org/docs/history.html

## 安装jdk

官网地址：https://www.oracle.com/java/technologies/downloads/#java8

查看系统架构

```bash
[root@node1.local ~]# uname -m
x86_64

```

下载jdk
![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/8f2f6c1dc2564f79a48fefe0e082ee3f.png)



创建jdk存放目录

```bash
[root@node1.local jdk]# mkdir /usr/local/src/jdk

[root@node1.local jdk]# ll
total 144896
drwxr-xr-x 2 root root      4096 Feb 10 13:11 ./
drwxr-xr-x 4 root root      4096 Feb 10 11:44 ../
-rw-r--r-- 1 root root 148362647 Nov 18 22:08 jdk-8u431-linux-x64.tar.gz

解压tar包
[root@node1.local jdk]# tar xf jdk-8u431-linux-x64.tar.gz 

[root@node1.local jdk1.8.0_431]# pwd
/usr/local/src/jdk/jdk1.8.0_431

```

配置环境变量

```bash
[root@node1.local ~]# vim .bashrc
export JAVA_HOME=/usr/local/src/jdk/jdk1.8.0_441
export PATH=$PATH:$JAVA_HOME/bin

[root@node1.local ~]# source .bashrc

```

测试

```bash
[root@node1.local ~]# java -version
java version "1.8.0_431"
Java(TM) SE Runtime Environment (build 1.8.0_431-b10)
Java HotSpot(TM) 64-Bit Server VM (build 25.431-b10, mixed mode)

```



## 安装maven

官方地址：https://maven.apache.org/download.cgi

创建maven存放目录

```bash
[root@node1.local maven]# mkdir /usr/local/src/maven

解压tar包
[root@node1.local maven]# tar xf apache-maven-3.9.9-bin.tar.gz
```

配置maven仓库，设置阿里镜像仓库，一定要配置一下，国内的下载jar快些，首先进入cd apache-maven-3.6.3目录，创建仓库存储目录ck

```bash
[root@node1.local maven]# cd apache-maven-3.9.9/

创建仓库存储目录ck
[root@node1.local apache-maven-3.9.9]# mkdir ck
[root@node1.local apache-maven-3.9.9]# ll
total 60
drwxr-xr-x 7 root root  4096 Feb 10 11:26 ./
drwxr-xr-x 3 root root  4096 Feb 10 11:25 ../
drwxr-xr-x 2 root root  4096 Feb 10 11:25 bin/
drwxr-xr-x 2 root root  4096 Feb 10 11:25 boot/
drwxr-xr-x 2 root root  4096 Feb 10 11:26 ck/
drwxr-xr-x 3 root root  4096 Feb 10 11:32 conf/
drwxr-xr-x 4 root root  4096 Feb 10 11:25 lib/
-rw-r--r-- 1 root root 18920 Aug 14 16:48 LICENSE
-rw-r--r-- 1 root root  5034 Aug 14 16:48 NOTICE
-rw-r--r-- 1 root root  1279 Aug 14 16:48 README.txt

```

进入cd conf目录，编辑 vi settings.xml文件，找到·localRepository下面复制一行加上<localRepository>/usr/local/apache-maven-3.6.3/ck</localRepository>， 在找到mirror 加上阿里的仓库配置，配置完成报错退出

```bash
[root@node1.local apache-maven-3.9.9]# cd conf/
[root@node1.local conf]# ll
total 28
drwxr-xr-x 3 root root  4096 Feb 10 11:32 ./
drwxr-xr-x 7 root root  4096 Feb 10 11:26 ../
drwxr-xr-x 2 root root  4096 Aug 14 16:48 logging/
-rw-r--r-- 1 root root 10473 Feb 10 11:32 settings.xml
-rw-r--r-- 1 root root  3645 Aug 14 16:48 toolchains.xml

[root@node1.local conf]# vim settings.xml 
....
#配置仓库存储目录位置
<localRepository>/usr/local/src/maven/apache-maven-3.9.9/ck</localRepository>
....
```

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/990e8fa0a25f4b05a457c35dd225f80d.png)




阿里的仓库配置

```bash
[root@node1.local conf]# vim settings.xml 
....
<mirror>
      <id>alimaven</id>
      <name>aliyun maven</name>
       <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>
</mirror>
....
```

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/ec49860c01884246889c8eb43037745e.png)





配置maven环境变量

```bash
[root@node1.local ~]# vim .bashrc
export MAVEN_HOME=/usr/local/src/maven/apache-maven-3.9.9
export PATH=$PATH:$MAVEN_HOME/bin

[root@node1.local ~]# source .bashrc
```

测试

```bash
[root@node1.local ~]# mvn -v
Apache Maven 3.9.9 (8e8579a9e76f7d015ee5ec7bfcdc97d260186937)
Maven home: /usr/local/src/maven/apache-maven-3.9.9
Java version: 1.8.0_431, vendor: Oracle Corporation, runtime: /usr/local/src/jdk/jdk1.8.0_431/jre
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "6.8.0-52-generic", arch: "amd64", family: "unix"

```

## 安装nodejs

```bash
[root@node1.local RuoYi-Cloud-master]# apt install npm
```

测试

```bash
[root@node1.local RuoYi-Cloud-master]# node -v
v18.19.1
[root@node1.local RuoYi-Cloud-master]# npm -v
9.2.0
```

## 安装mysql

```bash
[root@node1.local ~]# apt install mysql-server

[root@node1.local ~]# mysql -V
mysql  Ver 8.0.41-0ubuntu0.24.04.1 for Linux on x86_64 ((Ubuntu))
```

加入开机自启

```bash
[root@node1.local apt]# systemctl enable mysql.service
Synchronizing state of mysql.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable mysql
```

查看所有用户信息

```mysql
select user,host from mysql.user;
```

创建root账号，密码123456

```mysql
mysql> create user root @'%' identified by '123456';
```

用户所有权限但是不包括授权的权限

```mysql
mysql> grant all on *.* to root@'%'; 
```

给用户可以授权其他用户的权限

```mysql
mysql> grant Grant option on *.* to root@'%';
```

修改监听地址
![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/e6bd7269c87448888bc7f55f763f0404.png)



**注意**：配置8.0版本参考：我这里通过这种方式没有实现所有IP都能访问；我是通过直接修改配置文件才实现的，MySQL8.0版本把配置文件 `my.cnf` 拆分成`mysql.cnf `和`mysqld.cnf`，我们需要修改的是`mysqld.cnf`文件：

```bash
[root@node1.local mysql]# vim /etc/mysql/mysql.conf.d/mysqld.cnf
...
bind-address            = 0.0.0.0
mysqlx-bind-address     = 0.0.0.0
```

重启服务

```bash
[root@node1.local mysql.conf.d]# systemctl restart mysql
```

查看

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/915127f0e9dd429c870b83b1e4e0e283.png)


## 安装redis

```bash
[root@node1.local apt]# apt install redis
```

加入开机自启

```bash
[root@node1.local apt]# systemctl list-units --type=service | grep redis
  redis-server.service                   loaded active     running      Advanced key-value store
  
[root@node1.local apt]# systemctl enable redis-server.service --now
Synchronizing state of redis-server.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Executing: /usr/lib/systemd/systemd-sysv-install enable redis-server
```

## 安装nacos

先部署微服务注册中心Nacos，在部署微服务层各个模块，最后接入前端UI

官方地址：https://nacos.io/zh-cn/

下载tar包

```bash
[root@node1.local ~]# wget https://github.com/alibaba/nacos/releases/download/2.1.1/nacos-server-2.1.1.tar.gz
```

解压tar包

```bash
[root@node1.local ~]# tar -xf nacos-server-2.1.1.tar.gz && cd nacos
```

编辑配置文件

```bash
[root@node1.local ~]# cd ./nacos/conf/
[root@node1.local conf]# ll
total 96
drwxr-xr-x 2  502 staff  4096 Feb 10 16:20 ./
drwxr-xr-x 5 root root   4096 Feb 10 16:03 ../
-rw-r--r-- 1  502 staff  1224 May 11  2022 1.4.0-ipv6_support-update.sql
-rw-r--r-- 1  502 staff  9798 Feb 10 16:20 application.properties
-rw-r--r-- 1  502 staff  9754 Aug  8  2022 application.properties.example
-rw-r--r-- 1  502 staff   670 Aug  8  2022 cluster.conf.example
-rw-r--r-- 1  502 staff 31156 Aug  8  2022 nacos-logback.xml
-rw-r--r-- 1  502 staff 10825 Aug  8  2022 nacos-mysql.sql
-rw-r--r-- 1  502 staff  8939 Aug  8  2022 schema.sql

[root@node1.local conf]# vim application.properties
###nacos的默认访问路径（不需要修改）
server.servlet.contextPath=/nacos

#nacos默认端口（不需要修改）
server.port=8848

###使用MySQL作为数据源（取消注释）
spring.datasource.platform=mysql

###数据库数量（取消注释）
db.num=1

###连接数据库的信息（连接本机数据库地址不需要修改）
###JDBC连接数据库最后面添加 &allowPublicKeyRetrieval=true 允许进行SSL连接，MySQL高版本不开启该参数会导致SSL连接时出现错误，Nacos无法启动
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true

###连接数据库的用户名（需要指定用户名）
db.user.0=root

###连接数据库的用户密码（需要指定用户名密码）
db.password.0=123456
```

在MySQL中创建`ry-cloud`数据库

登录数据库

```mysql
[root@node1.local conf]# mysql -uroot -p123456
```

创建ry-cloud库

```mysql
mysql> create database `ry-cloud` charset utf8;
```

创建ry-config库

```mysql
mysql> create database `ry-config` charset utf8;
```

查看所有库

```mysql
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| ry-cloud           |
| ry-config          |
| sys                |
+--------------------+
6 rows in set (0.01 sec)
```

导入数据

```bash
[root@node1.local sql]# pwd
/root/RuoYi-Cloud/RuoYi-Cloud-master/sql

[root@node1.local sql]# ll
total 100
drwxr-xr-x  2 root root  4096 Jan  7 10:55 ./
drwxr-xr-x 13 root root  4096 Jan  7 10:55 ../
-rw-r--r--  1 root root 11985 Jan  7 10:55 quartz.sql
-rw-r--r--  1 root root 56722 Jan  7 10:55 ry_20240629.sql
-rw-r--r--  1 root root 20053 Jan  7 10:55 ry_config_20240902.sql
-rw-r--r--  1 root root  3083 Jan  7 10:55 ry_seata_20210128.sql

导入数据
[root@node1.local sql]# mysql -uroot -p123456 ry-config <ry_config_20240902.sql
mysql: [Warning] Using a password on the command line interface can be insecure.
[root@node1.local sql]# mysql -uroot -p123456 ry-cloud < ry_20240629.sql
mysql: [Warning] Using a password on the command line interface can be insecure.
```

修改Nacos的`application.properties`文件，将默认的Nacos库指向新的`ry-config`库，这样Nacos就可以管理项目的配置文件

```mysql
vim nacos/conf/application.properties
....
db.url.0=jdbc:mysql://192.1.7.244:3306/ry-config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
db.user.0=root
db.password.0=123456
```

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/bc81d0d7092c47c6b8e916e5a4fbbd6e.png)


启动 Nacos 服务（standalone表示单机模式运行，非集群模式）

```bash
[root@node1.local bin]# pwd
/root/nacos/bin

[root@node1.local bin]# ./startup.sh -m standalone
/usr/local/src/jdk/jdk1.8.0_431/bin/java -Djava.ext.dirs=/usr/local/src/jdk/jdk1.8.0_431/jre/lib/ext:/usr/local/src/jdk/jdk1.8.0_431/lib/ext  -Xms512m -Xmx512m -Xmn256m -Dnacos.standalone=true -Dnacos.member.list= -Xloggc:/root/nacos/logs/nacos_gc.log -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -Dloader.path=/root/nacos/plugins/health,/root/nacos/plugins/cmdb,/root/nacos/plugins/selector -Dnacos.home=/root/nacos -jar /root/nacos/target/nacos-server.jar  --spring.config.additional-location=file:/root/nacos/conf/ --logging.config=/root/nacos/conf/nacos-logback.xml --server.max-http-header-size=524288
nacos is starting with standalone
nacos is starting，you can check the /root/nacos/logs/start.out
```

查看日志

```bash
[root@node1.local bin]# tail -f 100 /root/nacos/logs/start.out
```

访问

```bash
地址：http://192.1.7.244:8848/nacos/index.html
账号：nacos
密码：nacos
```

此时Nacos已经获取到RuoYi-Cloud各组件的配置文件，后续修改各组件的配置文件，在Nacos中修改即可。

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/220a52f5c6984b3b86ec2b609b3a6bca.png)




## 打包RuoYi Cloud项目

进入RuoYi-Cloud目录打包项目

```bash
[root@node1.local ~]# cd RuoYi-Cloud
```

使用maven打包

```bash
[root@node1.local RuoYi-Cloud-master]# mvn clean package -Dmaven.test.skip=true
....
[INFO] ------------------------------------------------------------------------
[INFO] Reactor Summary for ruoyi 3.6.5:
[INFO] 
[INFO] ruoyi .............................................. SUCCESS [  8.096 s]
[INFO] ruoyi-common ....................................... SUCCESS [  0.002 s]
[INFO] ruoyi-common-core .................................. SUCCESS [01:35 min]
[INFO] ruoyi-api .......................................... SUCCESS [  0.002 s]
[INFO] ruoyi-api-system ................................... SUCCESS [  0.233 s]
[INFO] ruoyi-common-redis ................................. SUCCESS [  5.422 s]
[INFO] ruoyi-common-security .............................. SUCCESS [  0.772 s]
[INFO] ruoyi-auth ......................................... SUCCESS [ 58.992 s]
[INFO] ruoyi-gateway ...................................... SUCCESS [ 17.479 s]
[INFO] ruoyi-visual ....................................... SUCCESS [  0.001 s]
[INFO] ruoyi-visual-monitor ............................... SUCCESS [  6.823 s]
[INFO] ruoyi-common-datasource ............................ SUCCESS [  9.523 s]
[INFO] ruoyi-common-datascope ............................. SUCCESS [  0.121 s]
[INFO] ruoyi-common-log ................................... SUCCESS [  0.191 s]
[INFO] ruoyi-common-swagger ............................... SUCCESS [  1.419 s]
[INFO] ruoyi-modules ...................................... SUCCESS [  0.001 s]
[INFO] ruoyi-modules-system ............................... SUCCESS [  1.307 s]
[INFO] ruoyi-modules-gen .................................. SUCCESS [  1.825 s]
[INFO] ruoyi-modules-job .................................. SUCCESS [  1.882 s]
[INFO] ruoyi-modules-file ................................. SUCCESS [ 13.886 s]
[INFO] ruoyi-common-seata ................................. SUCCESS [ 24.541 s]
[INFO] ruoyi-common-sensitive ............................. SUCCESS [  0.109 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  04:28 min
[INFO] Finished at: 2025-02-10T18:26:29+08:00
[INFO] ------------------------------------------------------------------------

```

如果打包其中的一个项目，可以使用这种方式：

```bash
mvn package -Dmaven.test.skip=true -pl ruoyi-modules/ruoyi-system/ -am
```

# 四、部署模块

## 部署system系统模块

登录到nacos修改`ruoyi-system-dev.yml`文件，需要指定连接数据库

```bash
# spring配置
spring:
  redis:
    # 定义连接Redis的主机(默认本机不需要改)
    host: localhost
    port: 6379
    password:
      datasource:
         # 主库数据源,连接MySQL数据库的用户名和密码需要修改
          master:
            driver-class-name: com.mysql.cj.jdbc.Driver
            url: jdbc:mysql://192.1.7.244:3306/ry-cloud?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=true&serverTimezone=GMT%2B8
            username: root
            password: 123456
```

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/a55bb6d34e20443cb0e90096e0d616f6.png)


添加配置后，点击右下角的【发布】即可。

启动ruoyi-system模块

```bash
java -Dspring.profiles.active=dev \
-Dspring.cloud.nacos.config.file-extension=yml \
-Dspring.cloud.nacos.discovery.server-addr=192.1.7.244:8848 \
-Dspring.cloud.nacos.config.server-addr=192.1.7.244:8848 \
-jar /root/RuoYi-Cloud/RuoYi-Cloud-master/ruoyi-modules/ruoyi-system/target/ruoyi-modules-system.jar &> /var/log/ruoyi-system.log &
```

命令解释：

```bash
java	                               	     //Java 命令
-Dspring.profiles.active       //在spring中用于设置程序的配置环境（dev开发，test测试，prod生产）
-Dspring.cloud.nacos.config.file-extension 	//指定了程序的配置文件在Nacos的扩展名为yml
-Dspring.cloud.nacos.discovery.server-addr 	//指定了Nacos服务发现的地址和端口
-Dspring.cloud.nacos.config.server-addr    	//指定了Nacos配置中心的地址和端口
-jar   	                                   			//指定jar包路径和名称
```

查看日志

```bash
[root@node1.local RuoYi-Cloud-master]# tail -f /var/log/ruoyi-system.log
```

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/689885a37e874b45924815531e471310.png)


nacos查看服务，这时候可以看见服务已经注册上来了

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/914fc6e1a1914943b8045ba6bf8dff42.png)


![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/ac7fa1990b054658bb46c0b82e3d5727.png)


注册成功后，网关就可以通过Nacos获取到该服务的信息，来调用该服务。

## 部署auth认证模块

登录Nacos修改`ruoyi-auth-dev.yml`文件，指定连接Redis信息

```bash
spring:
  redis:
    host: localhost
    port: 6379
    password:
```

添加配置后，点击右下角的【发布】即可。

启动ruoy-auth模块

```bash
java -Dspring.profiles.active=dev \
-Dspring.cloud.nacos.config.file-extension=yml \
-Dspring.cloud.nacos.discovery.server-addr=192.1.7.244:8848 \
-Dspring.cloud.nacos.config.server-addr=192.1.7.244:8848 \
-jar /root/RuoYi-Cloud/RuoYi-Cloud-master/ruoyi-auth/target/ruoyi-auth.jar &> /var/log/ruoyi-auth.log &

```

查看日志

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/b22181b65b3946c692cbad2006bb2236.png)


在Nacos服务列表中查看是否注册成功

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/2ddb250af3d843e4b574bc7ddd741dde.png)


## 部署gateway网关模块

登录Nacos修改`ruoyi-gateway-dev.yml`文件，需要指定连接Redis信息

```bash
spring:
  redis:
    host: localhost
    port: 6379
    password:
```

添加配置后，点击右下角的【发布】即可。

启动ruoyi-gateway模块

```bash
java -Dspring.profiles.active=dev \
-Dspring.cloud.nacos.config.file-extension=yml \
-Dspring.cloud.nacos.discovery.server-addr=192.1.7.244:8848 \
-Dspring.cloud.nacos.config.server-addr=192.1.7.244:8848 \
-jar /root/RuoYi-Cloud/RuoYi-Cloud-master/ruoyi-gateway/target/ruoyi-gateway.jar &> /var/log/ruoyi-gateway.log &
```

查看日志

```bash
[root@node1.local target]# tail -f /var/log/ruoyi-gateway.log
```

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/cf55971de75042b7aa0ffeeb3037fdd9.png)


在Nacos服务列表中查看是否注册成功

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/124b02e0bc564e64882d702ec0faed8c.png)


## 部署monitor监控模块

monitor配置信息存储在nacos中，文件为ruoyi-monitor-dev.yml

```bash
# spring
spring:
  security:
    user:
      name: admin
      password: 123456
  boot:
    admin:
      ui:
        title: 若依服务状态监控
```

启动ruoyi-monitor模块

```bash
java -Dspring.profiles.active=dev \
-Dspring.cloud.nacos.config.file-extension=yml \
-Dspring.cloud.nacos.discovery.server-addr=192.1.7.244:8848 \
-Dspring.cloud.nacos.config.server-addr=192.1.7.244:8848 \
-jar /root/RuoYi-Cloud/RuoYi-Cloud-master/ruoyi-visual/ruoyi-monitor/target/ruoyi-visual-monitor.jar &> /var/log/ruoyi-monitor.log &
```

查看日志

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/30e63223cfb04923960d858f69232082.png)


在Nacos服务列表中查看是否注册成功

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/1ba815c29a734843b380861fa45ba386.png)


访问ruoyi-monitor的页面

```bash
url: http://192.1.7.244:9100/login
账号：admin
密码：123456
```

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/fe2227a0a4b74ae2be1b82fddf3c4af1.png)


## 部署ui前端

修改ruoyi-ui连接gateway的地址（本环境网关与ruoyi-ui部署在同一节点，无需修改）

```bash
[root@node1.local RuoYi-Cloud-master]#  vim RuoYi-Cloud/ruoyi-ui/vue.config.js
#...
  devServer: {
    host: '0.0.0.0',
    port: port,
    open: true,
    proxy: {
      // detail: https://cli.vuejs.org/config/#devserver-proxy
      [process.env.VUE_APP_BASE_API]: {
        target: `http://localhost:8080`,  #连接网关地址及端口
```

切换到UI目录

```bash
[root@node1.local RuoYi-Cloud-master]# cd ruoyi-ui/
```

指定依赖库地址，需要先下载项目的依赖

```bash
[root@node1.local ruoyi-ui]# npm install --registry=https://registry.npmmirror.com
```

构建项目

```bash
[root@node1.local ruoyi-ui]# npm run build:prod
```

**如果构建项目时报错**：error:0308010C:digital envelope routines::unsupported

主要是因为 nodeJs V17 版本发布了 OpenSSL3.0 对算法和秘钥大小增加了更为严格的限制，nodeJs v17 之前版本没影响，但 V17 和之后版本会出现这个错误。

输入以下以下命令，强制Node.js使用旧版本的OpenSSL库，以解决与新版本Node.js中加密库不兼容的问题。

```bash
export NODE_OPTIONS=--openssl-legacy-provider
```

编译打包项目

```bash
[root@node1.local ruoyi-ui]# npm run build:prod
```

打包成功后，会生成一个名为 dist 目录，里边就是前端页面文件，将页面文件放到nginx对应的发布项目目录（比如html目录）即可。

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/2e7d46f968c04841b30f6e5d64f3444c.png)


**部署Nginx发布UI（就在当前节点部署即可）**

```bash
[root@node1.local dist]# apt install nginx -y
```

准备UI站点配置文件

```bash
[root@node1.local conf.d]# vim /etc/nginx/conf.d/ruoyi.conf 
server {
    listen       80;
    server_name  localhost;

        location / {
            root   /usr/share/nginx/ruoyi-ui;
            try_files $uri $uri/ /index.html;
            index  index.html index.htm;
        }

        location /prod-api/{
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header REMOTE-HOST $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://localhost:8080/;
        }

        # 避免actuator暴露
        if ($request_uri ~ "/actuator") {
            return 403;
        }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
```

创建目录拷贝/dist目录下代码

```bash
[root@ruoyi-demo ruoyi-ui]# mkdir /usr/share/nginx/ruoyi-ui
[root@ruoyi-demo ruoyi-ui]# cp -r dist/* /usr/share/nginx/ruoyi-ui
```

启动Nginx

```bash
[root@node1.local conf.d]# systemctl enable nginx --now
```

Windows配置本地解析

```bash
C:\Windows\System32\drivers\etc\hosts
192.1.7.244 cherry.nuoyi.com
```

访问

```bash
url：http://cherry.nuoyi.com/login?redirect=%2Findex
账号：admin
密码：admin123
```

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/2d6fcdae20cc4f3fa3a5f986251df64e.png)


# 五、更新特定模块

更新ruoyi-auth模块，修改登录时的提示信息

```bash
[root@node1.local RuoYi-Cloud-master]# vim /root/RuoYi-Cloud/ruoyi-auth/src/main/java/com/ruoyi/auth/service/SysLoginService.java
#...
        {
            recordLogService.recordLogininfor(username, Constants.LOGIN_FAIL, "登录用户不存在");
            throw new ServiceException("输入的用户名：" + username + " 不存在");
        }
```

重新打包ruoyi-auth模块

```bash
命令说明：
-pl		//需要构建的目录
-am		//同时构建该模块依赖的其他模块
```

```bash
[root@node1.local RuoYi-Cloud-master]# cd RuoYi-Cloud/
mvn package -Dmaven.test.skip=true -pl ruoyi-auth -am
===========================================================
[INFO] ruoyi .............................................. SUCCESS [  1.266 s]
[INFO] ruoyi-common ....................................... SUCCESS [  0.001 s]
[INFO] ruoyi-common-core .................................. SUCCESS [  1.962 s]
[INFO] ruoyi-api .......................................... SUCCESS [  0.001 s]
[INFO] ruoyi-api-system ................................... SUCCESS [  0.051 s]
[INFO] ruoyi-common-redis ................................. SUCCESS [  0.128 s]
[INFO] ruoyi-common-security .............................. SUCCESS [  0.056 s]
[INFO] ruoyi-auth ......................................... SUCCESS [  3.560 s]
```

**提示：**同一个项目中，如果其他服务依赖于被更新的服务， 那么在更新这个服务时，也会重新打包和部署那些相关的服务。

停止ruoyi-auth重新运行

jps命令是java的命令行工具，用于列出正在运行的Java进程信息 

```bash
[root@node1.local RuoYi-Cloud-master]# jps
315905 ruoyi-modules-system.jar
318947 Jps
212244 nacos-server.jar
303327 ruoyi-visual-monitor.jar
301785 ruoyi-gateway.jar
301082 ruoyi-auth.jar

[root@node1.local RuoYi-Cloud-master]# kill 301082
```

重新启动ruoyi-auth模块

```bash
java -Dspring.profiles.active=dev \
-Dspring.cloud.nacos.config.file-extension=yml \
-Dspring.cloud.nacos.discovery.server-addr=192.1.7.244:8848 \
-Dspring.cloud.nacos.config.server-addr=192.1.7.244:8848 \
-jar /root/RuoYi-Cloud/RuoYi-Cloud-master/ruoyi-auth/target/ruoyi-auth.jar &> /var/log/ruoyi-auth.log &
```

登录ruoyi页面，输入一个不存在的用户来验证更新结果
![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/7593fcbf6cff4ce99620601c07370148.png)