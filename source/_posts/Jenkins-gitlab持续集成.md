---
title: Jenkins+gitlab持续集成
date: 2025-04-18 10:51:51
tags: CICD
categories: CICD
---
 之前我们已经部署了k8s和harbor仓库，本次来部署下jenkins和gitlab实现持续集成。

**概念**

持续集成是开发团队通过将代码的不同部分集成到共享存储库中，并频繁地进行构建和测试，以确保代码的一致性和稳定性。

**流程**
 在现在的开发模式中，一般的项目，协同开发是离不开的，这就涉及到多个开发人员编写处理自己负责的功能模块或者某些开发人员共同负责一个模块。于是，通过版本控制系统可以将各个开发人员的代码集成在该共享存储库里，在存储库里，每个开发人员根据需求的不同来创建对应的分支，在完成需求后，每个人都需要提交合并将开发分支代码集成在一起，这就需要解决代码冲突，并且如何除了code review之外如何确保这些更改对应用没有产生影响，一旦提交请求合并到主分支，自动化构建工具就会根据流程自动编译构建安装应用，并执行单元测试框架的自动化测试来校验提交的修改。

## 机器规划如下

| 服务器名称 | IP地址     | 配置                    |
| ---------- | ---------- | ----------------------- |
| jenkins211 | 10.0.0.211 | 2核，4GiB，系统盘 20GiB |
| gitlab212  | 10.0.0.212 | 4核，4GiB，系统盘 20GiB |

## 一、部署jenkins

**1.下载jenkins**

```bash
https://mirrors.jenkins-ci.org/debian/
```

**2.准备jenkins部署环境**

```bash
1 部署Jenkins的秘钥
[root@jenkins211 ~]# sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
    https://pkg.jenkins.io/debian/jenkins.io-2023.key

2 添加Jenkins的存储库
[root@jenkins211 ~]# echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
    https://pkg.jenkins.io/debian binary/ | sudo tee \
    /etc/apt/sources.list.d/jenkins.list > /dev/null

3 安装JRE环境(在线安装很慢，一般提前下载好jdk)
[root@jenkins211 ~]# sudo apt-get update
  sudo apt-get install fontconfig openjdk-17-jre
  sudo apt-get install jenkins  

4 创建jdk安装目录，解压到指定目录
[root@jenkins211 ~]# mkdir -pv /softwares
[root@jenkins211 ~]# tar xf jdk-17_linux-x64_bin.tar.gz -C /softwares/


5 配置jdk环境变量
[root@jenkins211 ~]# vim .bashrc
...
export JAVA_HOME=/softwares/jdk-17.0.8
export PATH=$PATH:$JAVA_HOME/bin

[root@jenkins211 ~]# source .bashrc 

6 验证jdk是否生效
[root@jenkins211 ~]# java --version
openjdk 17.0.13 2024-10-15
OpenJDK Runtime Environment (build 17.0.13+11-Ubuntu-2ubuntu122.04)
OpenJDK 64-Bit Server VM (build 17.0.13+11-Ubuntu-2ubuntu122.04, mixed mode, sharing)
```

**3.部署jenkins**

```bash
安装jenkins
[root@jenkins211 ~]# dpkg -i jenkins_2.489_all.deb 
# apt-get install jenkins #这种在线方式安装较慢，不建议。

修改jenkins的启动脚本并重启服务
[root@jenkins211 ~]# vim /lib/systemd/system/jenkins.service
......
User=root
Group=root

...
Environment="JENKINS_HOME=/var/lib/jenkins"
# 找到上面一行后添加如下一行
Environment="JAVA_HOME=/oldboyedu/softwares/jdk-17.0.8"
```

**4.访问jenkins地址**

```bash
http://10.0.0.211:8080/login?from=%2F
```

![img](https://gitee.com/ljh00928/csdn/raw/master/img/e1515ec05a604141a51fb41866f3b4f8.png)
**5.查看密码**

```bash
[root@jenkins211 ~]# cat /var/lib/jenkins/secrets/initialAdminPassword
88259fc092c34ded983b6917872****d
```

**6.安装jenkins插件**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/fb5658f33a674aaf8f16a7198b9da4e5.png)**7.创建用户**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/105460e20071422b84d3861c09145a54.png)**好啦，就这样jenkins就简简单单部署完成了**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/5c919d64bb60497db248332c7ba4d376.png)

8.修改时区

```
[root@jenkins211 ~]# ln -svf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
'/etc/localtime' -> '/usr/share/zoneinfo/Asia/Shanghai'
```

![img](https://gitee.com/ljh00928/csdn/raw/master/img/b6d8f708686a4daf973f86a9fb8c4fcc.png)

9.修改jenkins软件源

```
[root@jenkins211 ~]# sed -i.bak 's#updates.jenkins.io/download#mirror.tuna.tsinghua.edu.cn/jenkins#g' /var/lib/jenkins/updates/default.json
```

10.修改搜索引擎

```
[root@jenkins211 ~]# sed -i 's#www.google.com#www.baidu.com#g' /var/lib/jenkins/updates/default.json
```

11.替换jenkins国内下载地址

![img](https://gitee.com/ljh00928/csdn/raw/master/img/2e48d1b82ed74ec0971cfbdf1214a888.png)

 拉到最下面替换url

> 将升级站点URL替换成国内镜像地址
>  https://mirror.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json

## ![img](https://gitee.com/ljh00928/csdn/raw/master/img/9ae3dd35c62e4146a54b59671f70bf2d.png)

12.重新登录jenkins

> http://10.0.0.211:8080/restart

## 二、部署gitlab

参考地址：https://about.gitlab.com/install/#ubuntu

1. 安装并配置必要的依赖项

```bash
apt-get update
apt-get install -y curl openssh-server ca-certificates tzdata perl
```

2.安装 Postfix

```bash
apt-get install -y postfix
```

3.添加 GitLab 包仓库并安装包

```bash
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash
```

4.安装 GitLab 包

```bash
#url替换成自己本机的ip地址
EXTERNAL_URL="http://10.0.0.212" apt-get install gitlab-ee

#也可以选择离线部署
https://packages.gitlab.com/gitlab/gitlab-ce
```

5.查看密码

```bash
[root@gitlab212 ~]# cat /etc/gitlab/initial_root_password
....
Password: 8Et747zZeK0xKnx4gJ03HP+5MQ00yv4v3rzjy1a***=
....
```

6.登录

```bash
账号：root
密码：查看生成的密码
```

![img](https://gitee.com/ljh00928/csdn/raw/master/img/a7d21a5248714edf8437ae6646f79ff3.png)

7.设置语言

点击偏好设置

![img](https://gitee.com/ljh00928/csdn/raw/master/img/ea6267f5b46a4537a65173d42916acad.png)

选择中文-->保存

![img](https://gitee.com/ljh00928/csdn/raw/master/img/2c004cf24a564d328fea29af1008bd8a.png)8.开启本地网络请求

![img](https://gitee.com/ljh00928/csdn/raw/master/img/aaf8e8f036024678a602c833150fc730.png)

## 三、gitlab集成jenkins

1.创建群组

![img](https://gitee.com/ljh00928/csdn/raw/master/img/9d9589fef0c24fde925ab1874fbd4a99.png)

2.新建项目

![img](https://gitee.com/ljh00928/csdn/raw/master/img/9fda0fb6a0e74337b433cc67a415c1ef.png)

![img](https://gitee.com/ljh00928/csdn/raw/master/img/81ea81153b4a47d985cb4f2d7c31835a.png)

3.添加ssh密钥

服务器和代码仓库连接方式有两种

第一种: 基于用户名和密码的方式连接

第二种: 基于SSH免秘钥方式(和命令行的SSH免秘钥相同)

我们使用基于免秘钥方式: 方便+安全性高

```
[root@gitlab212 ~]# ssh-keygen     #生成密钥对
Generating public/private ed25519 key pair.
Enter file in which to save the key (/root/.ssh/id_ed25519): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_ed25519
Your public key has been saved in /root/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:GMd0qYCfisI7YgnL9gaTOvhSg9VWYVOtJ/ZmClhYCPc root@gitlab212
The key's randomart image is:
+--[ED25519 256]--+
|  ...o+oo...     |
|   .oo++ .o      |
|   . =Eooo       |
|  . + ++= .      |
|.o.o +..S+       |
|+=+ o .   +      |
|=+=.   . +       |
|BB .    .        |
|++=.             |
+----[SHA256]-----+
[root@gitlab212 ~]# 
[root@gitlab212 ~]# ll .ssh/
total 16
drwx------ 2 root root 4096 Dec 17 07:57 ./
drwx------ 4 root root 4096 Dec 15 12:01 ../
-rw------- 1 root root    0 Jul 23 16:40 authorized_keys
-rw------- 1 root root  411 Dec 17 07:57 id_ed25519        #私钥
-rw-r--r-- 1 root root   96 Dec 17 07:57 id_ed25519.pub    #公钥
[root@gitlab212 ~]# cat .ssh/id_rsa.pub 
cat: .ssh/id_rsa.pub: No such file or directory
[root@gitlab212 ~]# cat .ssh/id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILkNNA0/prl380e574O6+cfOSOAbpeJl+zhEdS5e58 root@gitlab212
```

 4.将公钥贴近gitlab

![img](https://gitee.com/ljh00928/csdn/raw/master/img/e5a3ce30996d414287f7cf26d27d9e9c.png)

添加公钥

![img](https://gitee.com/ljh00928/csdn/raw/master/img/20fd10f4b6214e02937eeb464ae75025.png)4.推送代码

配置邮箱

```
git config --local user.name "liangjh"
git config --local user.email "3101306637@qq.com"
```

创建仓库

```
#上传代码到服务器
[root@gitlab212 project]# pwd
/root/project
[root@gitlab212 project]# ll
total 228
drwxr-xr-x 6 root root  4096 Dec 17 08:09 ./
drwx------ 5 root root  4096 Dec 17 08:06 ../
-rw-r--r-- 1 root root 16458 Jun 13  2019 about.html
-rw-r--r-- 1 root root 20149 Jun 13  2019 album.html
-rw-r--r-- 1 root root 19662 Jun 13  2019 article_detail.html
-rw-r--r-- 1 root root 18767 Jun 13  2019 article.html
-rw-r--r-- 1 root root 18913 Jun 13  2019 comment.html
-rw-r--r-- 1 root root 16465 Jun 13  2019 contact.html
drwxr-xr-x 2 root root  4096 Sep 19  2022 css/
drwxr-xr-x 7 root root  4096 Dec 17 08:07 .git/
drwxr-xr-x 5 root root  4096 Sep 19  2022 images/
-rw-r--r-- 1 root root 29627 Jun 29  2019 index.html
drwxr-xr-x 2 root root  4096 Sep 19  2022 js/
-rw-r--r-- 1 root root 24893 Jun 13  2019 product_detail.html
-rw-r--r-- 1 root root 20672 Jun 13  2019 product.html


git clone git@10.0.0.212:ops/project.git
cd project
git switch --create main

#提交全部代码到缓存区
[root@gitlab212 project]# git add .    

#提交到远程仓库
[root@gitlab212 project]# git commit -m "第一次提交"

#查看分支
[root@gitlab212 project]# git branch

#推送到master分支
[root@gitlab212 project]# git push -u origin master
```

5.gitlab查看

![img](https://gitee.com/ljh00928/csdn/raw/master/img/1d5f28d411084e1e9bd09a9e30a5c7ab.png)

## 四、配置jenkins和gitlab免密

1.jenkins生成密钥对

```
[root@jenkins211 ~]# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:8f0e/2XSZCdAGULp+LEkvVjeFo7YVJWv7HDFQDzfi1I root@jenkins211
The key's randomart image is:
+---[RSA 3072]----+
|         .o.o*o. |
|          .oo =  |
|        .+ ..  *.|
|        ooB..E  *|
|        S@.Boooo=|
|        o B.=o+*.|
|           ..++ +|
|             ..=.|
|              . +|
+----[SHA256]-----+
[root@jenkins211 ~]# 
[root@jenkins211 ~]# cat .ssh/id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCrTMTf0oqtW7GktgZBdhqkjCBp2xGCcGeEo8ZcEKaN352TDMGKRI5O+0gAwUBj9HMAkPASJsejtTQIrT5jkBuW/8L5RXPDqCCyuzz1nkW/rPiyVvuAvvkQxFMBxL4w/AoDAhcsxpV73DCyYTrjQ8Tj7PfSViEUKZFpPbtroOZeCgCVa/mNPj7hGI2xUapKd0rdiGpXlDMPlD1uN9TET1WTj157QbYbgtsA6y+K2hjG97HHta0miovC4vqbaKQuohbYThbkwE5v6Ndjg8WSYjXEO6SDgemR6/c/JIyMPv1rNUitzl6dg4jykJ2pCTsemvFL7Vdjypbn9l0bPK8cq+mz0W8ZKLd2Ijqshj67SFWjFxJs0/yabg3jdMTdnYxRYdiG2vaMlAmtgGbXAUGXMqCzXNftmSpyqlSxGhfKLUC76LCqc4nBhqt2reN3L5yd/rzVZENdpok8cdV/dZag8ibzlgeqOL60ONThYTPZTACyw+U2K2HhqLmoCVDpYxU= root@jenkins211
```

 2.**将公钥加入到gitlab**

![img](https://gitee.com/ljh00928/csdn/raw/master/img/d151e8f5f16c4650bd5a2135dfc61e60.png)

3.拉取测试

```
# 随便拉一个位置测试就行
[root@jenkins211 ~]# git clone git@10.0.0.212:ops/project.git
```

## 五、 jenkins推送代码到gitlab

1.jenkins创建项目

![img](https://gitee.com/ljh00928/csdn/raw/master/img/3bb3f56c2bbd4ab0a4d33b70589d33a6.png)

2.填写git地址![img](https://gitee.com/ljh00928/csdn/raw/master/img/9cd693ded1bf4da29b08b1ddab163628.png)

 3.jenkins构建

![img](https://gitee.com/ljh00928/csdn/raw/master/img/6caece5f65ef4edf9405091af52edf45.png)

>  构建完成之后，代码会拉取到jenkins工作目录中，在日志中查看工作目录的位置

 ![img](https://gitee.com/ljh00928/csdn/raw/master/img/860ce7b0d00f4b28b93dc091057d0fde.png)

 服务器查看，代码已经成功拉取到jenkins工作目录。

```
[root@jenkins211 test]# pwd
/var/lib/jenkins/workspace/test
[root@jenkins211 test]# ll
total 228
drwxr-xr-x 6 root root  4096 Dec 17 17:22 ./
drwxr-xr-x 3 root root  4096 Dec 17 17:22 ../
-rw-r--r-- 1 root root 16458 Dec 17 17:22 about.html
-rw-r--r-- 1 root root 20149 Dec 17 17:22 album.html
-rw-r--r-- 1 root root 19662 Dec 17 17:22 article_detail.html
-rw-r--r-- 1 root root 18767 Dec 17 17:22 article.html
-rw-r--r-- 1 root root 18913 Dec 17 17:22 comment.html
-rw-r--r-- 1 root root 16465 Dec 17 17:22 contact.html
drwxr-xr-x 2 root root  4096 Dec 17 17:22 css/
drwxr-xr-x 8 root root  4096 Dec 17 17:22 .git/
drwxr-xr-x 5 root root  4096 Dec 17 17:22 images/
-rw-r--r-- 1 root root 29627 Dec 17 17:22 index.html
drwxr-xr-x 2 root root  4096 Dec 17 17:22 js/
-rw-r--r-- 1 root root 24893 Dec 17 17:22 product_detail.html
-rw-r--r-- 1 root root 20672 Dec 17 17:22 product.html
```

4.配置webhook

jenkins需要安装gitlab插件

![img](https://gitee.com/ljh00928/csdn/raw/master/img/52f75e68a66f45c98744132684c782c7.png)

 勾选插件

![img](https://gitee.com/ljh00928/csdn/raw/master/img/d1ffaa4591104bc0b6c840b5420990ce.png)

 jenkins生成令牌

![img](https://gitee.com/ljh00928/csdn/raw/master/img/58cc8d038daa445dbc23612a4e0bc49b.png)

 gitlab添加令牌，填写jenkins url

![img](https://gitee.com/ljh00928/csdn/raw/master/img/276546fd9ab44bb58382021626f3b884.png)

gitlab测试钩子状态

![img](https://gitee.com/ljh00928/csdn/raw/master/img/962a11f4e4c1450d8cee1548aa69720a.png)

##  六、测试gitlab和jenkins持续集成

gitlab修改代码提交，这里我们直接加个p标签测试

```
[root@gitlab212 project]# vim about.html
<!DOCTYPE html>

<html xmlns="http://www.w3.org/1999/xhtml">

<head>

        <p> 第一次提交 </p>

    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="applicable-device" content="pc,mobile" />
    <title>关于我们</title>
```

提交代码到gitlab

```
git add .

git commit -m ＂第一次提交"

git push origin master
```

jenkins观察是否自动拉取，这时候jenkins检测到gitlab分支有变动，自动拉取代码到了jenkins工作目录中了~

![img](https://gitee.com/ljh00928/csdn/raw/master/img/81321be1610f45fd921b4341abf05b57.png)

```
[root@jenkins211 test]# cat about.html | head -10
?<!DOCTYPE html>

<html xmlns="http://www.w3.org/1999/xhtml">

<head>

	<p> 第一次提交 </p>

    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
```
