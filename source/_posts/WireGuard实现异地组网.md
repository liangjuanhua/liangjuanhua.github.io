---
title: WireGuard实现异地组网
date: 2025-04-16 15:27:30
tags: 网络篇  
categories: 网络篇
---
 WireGuard是一个易于配置、快速、安全的基于UDP协议的开源VPN软件。Wireguard具有自定义配置路由转发的能力，所以可以被用来在多个不同地域将设备所在的内网网络通过路由转发的方式串通起来，组建一张属于自己的大内网！有时候，我们想将本地计算机上提供的服务与小伙伴分享，但是我们既没有公网IP，又希望能够有足够的安全性，避免使其暴露在公网上。因此，我们基于Wireguard这一项最基本的特性，设计和实现了一套异地组网方案。我们基于Wireguard这一项最基本的特性，设计和实现了一套异地组网方案。这里给大家介绍两种部署方式，以供大家选择。

## 一、WireGuard 基本概念

首先使用 WireGuard 你需要在[系统](https://www.moyann.com/tag/系统/)中创建一块虚拟网卡，并配置好这个虚拟网卡的 IP [地址](https://www.moyann.com/tag/地址/)，掩码，网关不需要配置（可以使用 wg-quick@ 自动化）

然后你使用 WireGuard [连接](https://www.moyann.com/tag/连接/)另一台设备，两台互相 peer 对方并验证各自的公钥私钥是否正确，全部正确后成功建立 peer（可以使用 wg-quick@ 自动化）

建立成功后，所有前往虚拟网卡的[流量](https://www.moyann.com/tag/流量/)都将被重新封装后发往另一台设备，由另一台设备解封装然后得到数据报文并在内部查找路由并匹配报文目的地。（可以使用 wg-quick@ 自动化）

以上为建立一个 WireGuard VPN 链接的过程，建立好后，A 设备与 B 设备互相需要保证虚拟网卡的 IP 在相同网络位的地址段中，并且这个地址段被 WireGuard 的配置[文件](https://www.moyann.com/tag/文件/) AllowedIPs 所允许通过

如果你试图从 A 设备访问 B 设备的对端子网，你需要在 A 设备上配置系统路由，将系统三层网络的路由目的地指向对端虚拟 IP 地址，出接口为虚拟网卡，并且这个地址段必须被 WireGuard 的配置文件 AllowedIPs 所允许通过

最后，在 WireGuard 中的所有数据报文，都采用 UDP 的方式发送。
 

## 二、安装 WireGuard

1.安装Wireguard

```
#root权限
sudo -i

#安装wireguard软件
apt install wireguard resolvconf -y

#开启IP转发
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```

2.修改目录权限

```
cd /etc/wireguard/
chmod 0777 /etc/wireguard

#调整目录默认权限
umask 077
```

3.生成服务器密钥

```
#生成私钥
wg genkey > server.key

#通过私钥生成公钥
wg pubkey < server.key > server.key.pub
```

4.生成客户端密钥

```
#生成私钥
wg genkey > client1.key

#通过私钥生成公钥
wg pubkey < client1.key > client1.key.pub
```

5.查看密钥

```
cat server.key && cat server.key.pub && cat client1.key && cat client1.key.pub
```

6.创建服务端配置文件

```
echo "
[Interface]
PrivateKey = $(cat server.key) # 填写本机的privatekey 内容
Address = 10.0.8.1 #本机虚拟局域网IP

PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
#注意eth0需要为本机网卡名称

ListenPort = 50814 # 监听端口
DNS = 8.8.8.8
MTU = 1420
[Peer]
PublicKey =  $(cat client1.key.pub)  #自动client1的公钥
AllowedIPs = 10.0.8.10/32 #客户端所使用的IP" > wg0.conf
```

7.启动服务

```
#设置开机自启动
systemctl enable wg-quick@wg0

#启动wg0
wg-quick up wg0

#关闭wg0
wg-quick down wg0

#如果启动失败只有有wg0网卡，需删除再次启动
ip link delete wg0
```

8.配置客户端

```
Windows下载地址：https://www.wireguard.com/install/

#客户端配置
[Interface]
PrivateKey = 6M8HEZioew+vR3i53sPc64Vg40YsuMzh4vI1Lkc88Xo= #此处为client1的私钥
Address = 10.0.8.10 #此处为peer规定的客户端IP
MTU = 1500

[Peer]
PublicKey = Tt5WEa0Vycf4F+TTjR2TAHDfa2onhh+tY8YOIT3cKjI= #此处为server的公钥
AllowedIPs = 10.0.8.0/24 #此处为允许的服务器IP
Endpoint = 114.132.56.178:50814 #服务器对端IP+端口
```

9.连接

![4658e56cf5dc493194931b981480e24a.png](https://gitee.com/ljh00928/csdn/raw/master/img/4658e56cf5dc493194931b981480e24a.png)编辑这时候很明显我电脑和服务器地址不在一个网段，但是我也可以成功访问公司网络咯~

![09870e03be6c457aaca5f71da26dc80c.png](https://gitee.com/ljh00928/csdn/raw/master/img/09870e03be6c457aaca5f71da26dc80c.png)

## 三、docker部署

这种方式简单，前提是需要安装docker。关于docker部署，cicd专栏-kubeadm部署k8s一篇博文中有写到。感兴趣小伙伴可以参考

```
docker run -d \
  --name=wg-easy \
  -e WG_HOST=123.123.123.123 (🚨这里输入服务器的公网IP) \
  -e PASSWORD=passwd123 (🚨这里输入你的密码) \
  -e WG_DEFAULT_ADDRESS=10.0.8.x （🚨默认IP地址）\
  -e WG_DEFAULT_DNS=114.114.114.114 （🚨默认DNS）\
  -e WG_ALLOWED_IPS=10.0.8.0/24 （🚨允许连接的IP段）\
  -e WG_PERSISTENT_KEEPALIVE=25 （🚨重连间隔）\
  -v ~/.wg-easy:/etc/wireguard \
  -p 51820:51820/udp \
  -p 51821:51821/tcp \
  --cap-add=NET_ADMIN \
  --cap-add=SYS_MODULE \
  --sysctl="net.ipv4.conf.all.src_valid_mark=1" \
  --sysctl="net.ipv4.ip_forward=1" \
  --restart unless-stopped \
  weejewel/wg-easy
```

容器更新命令

```
docker stop wg-easy
docker rm wg-easy
docker pull weejewel/wg-easy
```

