---
title: harbor部署
date: 2025-04-18 10:48:42
tags: CICD
categories: CICD
---
###  一、安装docker

这里我们直接使用docker脚本安装，关于脚本上一篇博文有写，感兴趣的朋友可以参考一下

```bash
1.部署docker和docker-compose 
1.1 解压软件包
[root@harbor ~]# tar xf docker-docker-compose.tar.gz 

1.2 安装docker和docker-compose运行时
[root@harbor ~]# ./install-docker.sh i

1.3 查看版本 
[root@harbor ~]# docker --version
Docker version 20.10.24, build 297e128
[root@harbor ~]# 
[root@harbor ~]# docker-compose --version
Docker Compose version v2.23.0
```

### **二、安装harbor**

##### **准备harbor安装包**

```bash
 https://github.com/goharbor/harbor/tags
```

##### 创建证书的工作目录

```bash
[root@harbor ~]# mkdir -pv /softwares/harbor/certs/{ca,harbor-server,docker-client}
mkdir: created directory '/softwares/harbor/certs'
mkdir: created directory '/softwares/harbor/certs/ca'  # 将来存储的是自建CA证书
mkdir: created directory '/softwares/harbor/certs/harbor-server'  # harbor服务端使用的证书
mkdir: created directory '/softwares/harbor/certs/docker-client'  # harbor客户端链接时使用的证书文件
```

##### 解压软件包

```bash
[root@harbor ~]# tar xf harbor-offline-installer-v2.7.4.tgz -C /softwares/
```

##### 进入到harbor证书存放目录

```bash
[root@harbor ~]# cd /softwares/harbor/certs/
```

##### 生成自建CA证书

```bash
1.创建CA的私钥
[root@harbor certs]# pwd
/softwares/harbor/certs
[root@harbor certs]# openssl genrsa -out ca/ca.key 4096

2 基于自建的CA私钥创建CA证书(注意，证书签发的域名范围)
[root@harbor certs]# openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=cherry.com" \
 -key ca/ca.key \
 -out ca/ca.crt

3 查看自建证书信息
[root@harbor certs]# openssl  x509 -in ca/ca.crt -noout -text
```

##### 配置harbor证书

```bash
1.生成harbor服务器的私钥
[root@harbor certs]# openssl genrsa -out harbor-server/harbor.cherry.com.key 4096

2 harbor服务器基于私钥签发证书认证请求（csr文件），让自建CA认证
[root@harbor certs]# openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=harbor.cherry.com" \
    -key harbor-server/harbor.cherry.com.key \
    -out harbor-server/harbor.cherry.com.csr
    
3 生成 x509 v3 的扩展文件用于认证
cat > harbor-server/v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=harbor.cherry.com
EOF
             
4 基于 x509 v3 的扩展文件认证签发harbor server证书
[root@harbor certs]# openssl x509 -req -sha512 -days 3650 \
    -extfile harbor-server/v3.ext \
    -CA ca/ca.crt -CAkey ca/ca.key -CAcreateserial \
    -in harbor-server/harbor.cherry.com.csr \
    -out harbor-server/harbor.cherry.com.crt
```

##### 修改harbor的配置文件使用自建证书

```bash
注意所在路径
[root@harbor harbor]# pwd
/softwares/harbor

修改harbor的配置文件使用自建证书
[root@harbor harbor]# cp harbor.yml{.tmpl,}

编辑harbor配置文件
[root@harbor harbor]# vim harbor.yml
...
hostname: harbor.cherry.com
https:
  ...
  certificate: /softwares/harbor/certs/harbor-server/harbor.cherry.com.crt
  private_key: /softwares/harbor/certs/harbor-server/harbor.cherry.com.key
...
harbor_admin_password: 1
...
data_volume: /data/harbor
```

##### 安装harbor

```bash
[root@harbor harbor]# ./install.sh --with-chartmuseum
```

### 三、访问

##### Windows做hosts解析

```bash
编辑Windowshosts文件
位置  C:\Windows\System32\drivers\etc\hosts
...
10.0.0.250 harbor.cherry.com
```

##### 访问

```bash
https://harbor.cherry.com

账号：admin
密码：1
```

![img](https://gitee.com/ljh00928/csdn/raw/master/img/d7c3895e963d46c5afaf5f99865075d6.png)

