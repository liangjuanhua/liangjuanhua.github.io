---
title: 封装一个专属自己的Windows 系统
date: 2025-04-16 17:23:27
tags: Windows
categories: Windows
---
​     因为我们公司属于零售行业，每次有新的门店开店，我们都需要安装大量我们自己软件和打印机驱动等，为了简化这种繁琐的工作和统一化管理，我们选择封装自己的iso镜像。插入u盘，即可安装我们自定义的iso镜像，并实现开机自动激活操作系统。废话不多说，直接上教程。本次以vmware虚拟机为案例，实体机操作也大差不差。

# 流程

- 准备干净系统
- 加入需要的软件
- 实现开机自动激活Windows
- dism生成install.wim文件
- anyburn替换install.wim文件
- anyburn生成新的iso镜像
- 烧录镜像到u盘

# **一、准备Windows系统**

1.虚拟机添加硬盘

![img](https://gitee.com/ljh00928/csdn/raw/master/img/44500211ea6b406e89329a537e0479ac.png)

2.Windows添加磁盘

![img](https://gitee.com/ljh00928/csdn/raw/master/img/4e908b5765384ef6a77cdcb1b5c24dd0.png)

3.查看卷 

![img](https://gitee.com/ljh00928/csdn/raw/master/img/8f0680eb1ab14ce8ad7a07706026567e.png)

4.加入自己需要封装的软件 

5.关机，按F2进入biso，修改启动项CD-ROM为第一启动项【实体机修改UEFI为U盘启动或者直接进PE界面操作】

![img](https://gitee.com/ljh00928/csdn/raw/master/img/21ecbdf8e10b4317b5eb4d047bed1b5c.png)

 6.按照以下步骤新增启动文件，也可以选择进入PE界面制作制作.wim启动文件。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/71279447b14f40a18dbb6c4bd9359c9c.png)

![img](https://gitee.com/ljh00928/csdn/raw/master/img/34b638cf3c0743b0aa52d169981da9ab.png)

![img](https://gitee.com/ljh00928/csdn/raw/master/img/d3db2fcbcf4b4e8a86821c9dacb8866b.png)

![img](https://gitee.com/ljh00928/csdn/raw/master/img/05ce4b65f54449709a99756f42382746.png)

7.使用dism生成install.wim文件

这里系统盘是E盘

![img](https://gitee.com/ljh00928/csdn/raw/master/img/a0d638875ef04f4a8930c20c176aba0d.png)

 8.生成.win文件

```
dism /capture-image /imagefile:D:\install.wim /capturedir:E:\ /Name:Windows10_kezhihua
```

- **`dism`**：调用 DISM 工具。
- **`/capture-image`**：指定要捕获一个映像文件（`install.wim`）。
- **`/imagefile:D:\install.wim`**：指定捕获的映像文件的保存路径和名称（在此示例中，保存到 `D:` 驱动器的 `install.wim` 文件中）。
- **`/capturedir:E:\`**：指定要捕获的源目录。`E:` 是源系统所在的驱动器或分区。
- **`/Name:Windows10_kezhihua`**：给捕获的映像文件指定一个名称（此例中，名称为 `Windows10_kezhihua`）。

![img](https://gitee.com/ljh00928/csdn/raw/master/img/aee6e64b9d774947bb7515da2fb82507.png)

 9.关闭命令行界面，点击继续

![img](https://gitee.com/ljh00928/csdn/raw/master/img/82a34ec080024db4848790c2a329a0ff.png)
10.查看生成的install.wim文件

![img](https://gitee.com/ljh00928/csdn/raw/master/img/5af42ac11a6b4fefbb7b686b2a540690.png)

# **二、实现Windows开机永久激活** 

使用数字许可证，加入Windows定时任务，开机执行。执行完成之后再自删。大概就是这个思路

参考地址：

https://github.com/TGSAN/CMWTAT_Digital_Edition/issues/81

直接使用脚本无法实现免交互自动激活，需要手动输入1

![image-20250416172651905](https://gitee.com/ljh00928/csdn/raw/master/img/image-20250416172651905.png)

修改脚本内容，实现免交互自动激活

![image-20250416172710159](https://gitee.com/ljh00928/csdn/raw/master/img/image-20250416172710159.png)

使用bat脚本开机自动激活

```bat
@echo off
:: 设置文件路径
set "activation_script=C:\Microsoft-Activation-Scripts-master\MAS\Separate-Files-Version\Activators\HWID_Activation.cmd"

:: 检查文件是否存在
if exist "%activation_script%" (
    echo Running HWID Activation script...
    :: 执行 HWID_Activation.cmd 文件
    call "%activation_script%"
) else (
    echo Error: HWID_Activation.cmd file not found!
)

:: 等待用户按键后退出
pause
```

加入Windows任务计划

![image-20250416172756740](https://gitee.com/ljh00928/csdn/raw/master/img/image-20250416172756740.png)

创建基本任务

![image-20250416172809175](https://gitee.com/ljh00928/csdn/raw/master/img/image-20250416172809175.png)

创建任务名称

![image-20250416172821507](https://gitee.com/ljh00928/csdn/raw/master/img/image-20250416172821507.png)

选择计算机启动时

![image-20250416172838445](https://gitee.com/ljh00928/csdn/raw/master/img/image-20250416172838445.png)

添加上面所写bat脚本路径

![image-20250416172849228](https://gitee.com/ljh00928/csdn/raw/master/img/image-20250416172849228.png)

完成

![image-20250416172902047](https://gitee.com/ljh00928/csdn/raw/master/img/image-20250416172902047.png)

因为脚本需要联网激活。所有更改计算机属性。选择联网时启动

![image-20250416172915961](https://gitee.com/ljh00928/csdn/raw/master/img/image-20250416172915961.png)

这样就实现了开机自动永久激活Windows操作系统

![image-20250416172929292](https://gitee.com/ljh00928/csdn/raw/master/img/image-20250416172929292.png)

# **三、编辑iso镜像**

这里我们使用anyburn工具，将生成的install.wim加入镜像里面

anyburn地址：[Download AnyBurn](https://www.anyburn.com/download.php)

编辑镜像

![img](https://gitee.com/ljh00928/csdn/raw/master/img/d249d2303afa4beca3337f6a12e8b51c.png)

在sources目录中添加install.wim（有些Windows官方镜像中包含install.esd文件，需要删除这个文件）

![img](https://gitee.com/ljh00928/csdn/raw/master/img/343bbd5335ce419a8d79219d26b3db70.png)

生成镜像

![img](https://gitee.com/ljh00928/csdn/raw/master/img/e67d15ab6f2549be85af906a9c6d2194.png)

# **四、烧录镜像** 

地址：[Rufus - Create bootable USB drives the easy way](https://rufus.ie/en/)

使用rufus烧录镜像，大于4G需要使用NTFS文件格式

![img](https://gitee.com/ljh00928/csdn/raw/master/img/d2008ae8a7f7452a91418a1812d40ab8.png)

实际工作中可以配合Windows的WDS服务,这个需要解决DHCP中继问题，实现远程一站式部署。

还可以使用Cobbler+kickstart配合，也可以实现全自动一站式远程装机，感兴趣小伙伴可以参考我这篇博文：[Cobbler+kickstart实现批量全自动装机_cobbler批量安装-CSDN博客](https://blog.csdn.net/m0_69326428/article/details/144848007)