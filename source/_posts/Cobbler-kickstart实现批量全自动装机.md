---
title: Cobbler+kickstart实现批量全自动装机
date: 2025-04-16 17:34:26
tags: Windows
categories: Windows
---
**cobbler简介**
　　cobbler 是一个系统启动服务boot server,可以通过pxe得方式用来快速安装，重装系统，支持安装不同linux发行版和windows。这个工具是用python开发，方便小巧，15k行代码，使用简单得命令完成pxe网络安装环境配置，还可以管理dhcp，dns，yum包镜像。cobbler可以命令行，也可以web（cobbler-web）,还提供api接口，可以方便二次开发使用。其实就是多安装树的pxe环境，是pxe的高级应用

**cobbler支持的功能**
       1、pxe支持

　　2、dhcp管理

　　3、dns服务管理（bind，dnsmasq）

　　4、电源管理

　　5、kickstart支持

　　6、yum仓库管理

　　7、tftp（pxe启动时需要）

　　8、apache，提供ks得安装源，并提供定制化得ks配置，同时，它和apache做了深度整合，通过cobbler，可以师兄redhat/centos/fedora系统得快速部署，同时也支持suse、debian（ubuntu）系统，通过配置开可以支持windows

**cobbler架构及工作原理、核心框架**
**cobbler工作原理**
![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/f7d7868539934ab9b74ae6fbe7762862.png)
cobbler框架
![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/8ebcfa6d523145e8afdc6f9cac9be923.png)
介绍一下profile核心，由三个组件组成

 - repositories （安装树或安装源）
   mirror 镜像，光盘或者网络中得安装源
    import 导入
 - distribution（vmlinuz-内核，initrd.img-引导映像文件）
   cobbler 自动从reporitories抽取出来生成
    kickstart file 组成得完完整整得系统发行版

cobbler就是较早pxe的升级版，优点容易配置，还自带web界面比较易于管理，但是中文资料少，（有人测试：cobbler不会应为在局域网中启动了dhcp而导致有些机器因为默认从pxe启动在重启服务器后加载tftp内容导致启动终止，这部分没有验证）

可以通过cobbler自动部署dhcp，tftp，http，在安装过程中加载ks无人值守安装应答文件实现无人值守，从客户端使用pxe引导启动安装

## 一、准备Windows的ADK和win PE

ADK下载地址：https://go.microsoft.com/fwlink/?linkid=2026036

win PE下载地址：https://go.microsoft.com/fwlink/?linkid=2022233

注意，adk 的两个都要下载，这俩都是引导包，真正的安装程序会由这俩软件进行下载。



## 二、安装 ADK 和 WinPE

![img](https://gitee.com/ljh00928/csdn/raw/master/img/aebd29209beb5504ca5f1580a2c8acc1.png)

![img](https://gitee.com/ljh00928/csdn/raw/master/img/602799dad7b31a1fd665eb8ee22940ac.png)



安装完后，以管理员身份打开部署和映像工具环境

![img](https://gitee.com/ljh00928/csdn/raw/master/img/175d96b3f6859cb5615d1eaaf152710d.png)



定制 Win 10 PE

```cmd
copype amd64 C:\winpe

Dism /mount-image /imagefile:C:\winpe\media\sources\boot.wim /index:1 /mountdir:C:\winpe\mount

echo net use z: \\192.168.0.253\share >> C:\winpe\mount\Windows\System32\startnet.cmd
echo z:\win\setup.exe /unattend:z:\win\win10_x64_bios_auto.xml >> C:\winpe\mount\Windows\System32\startnet.cmd

Dism /unmount-image /mountdir:C:\winpe\mount /commit
MakeWinPEMedia /ISO C:\winpe C:\winpe\winpe_win10_amd64.iso
```

1. 本地生成 winpe 文件目录
2. dism 挂载 winpe 的启动文件到 winpe 的 mount 目录
3. 将启动命令硬编码写死到 winpe 的 startnet.cmd 文件里
4. 无人值守安装
5. 卸载 winpe 的挂载（一定要执行，否则直接强制删除文件夹会出一些稀奇古怪的问题）
6. 制作 win10 镜像，名为 winpe_win10_amd64.iso



## 三、乌班图安装Cobbler

`乌班图安装需要编译安装，建议使用centos安装`

安装Apache

```bash
[root@node1.local ~]# apt update
[root@node1.local ~]# apt install apache2
```

启用所需的 Apache 模块

使用 `a2enmod` 来启用 `proxy`、`proxy_http` 和 `rewrite` 模块

```bash
a2enmod proxy
a2enmod proxy_http
a2enmod rewrite
```

检查是否正确启用模块

```bash
apache2ctl -M | grep proxy

proxy_module (shared)
proxy_http_module (shared)
```

创建 TFTP 根目录的符号链接

Cobbler 需要 TFTP 目录来进行 PXE 启动。创建一个符号链接，指向 `tftpboot` 目录

```bash
[root@node1.local ~]# ln -s /srv/tftp /var/lib/tftpboot
```

重新启动 Apache 服务

```bash
[root@node1.local ~]# systemctl restart apache2
```

查看服务状态

```bash
[root@node1.local ~]# systemctl status apache2
```

构建 `.deb` 包

下载 Cobbler 源代码

```bash
[root@node1.local ~]# git clone https://github.com/cobbler/cobbler.git
[root@node1.local ~]# cd cobbler
```

安装 `debuild` 和其他构建工具

```bash
[root@node1.local cobbler]# apt update
[root@node1.local cobbler]# apt install devscripts build-essential fakeroot debhelper
```

构建

```bash
[root@node1.local cobbler]# make debs
```

查看构建包位置

```bash
[root@node1.local ~]# find ~ -iname '*.deb'
/root/cobbler/deb-build/cobbler-tests-containers_3.4.0_all.deb
/root/cobbler/deb-build/cobbler_3.4.0_all.deb
/root/cobbler/deb-build/cobbler-tests_3.4.0_all.deb
/root/cobbler-tests-containers_3.4.0_all.deb
/root/cobbler_3.4.0_all.deb
/root/cobbler-tests_3.4.0_all.deb

```

安装

```bash
[root@node1.local ~]# dpkg -i /root/cobbler_3.4.0_all.deb

#安装的时候会提示以下缺少依赖
fence-agents
xorriso
python3-gunicorn
python3-pymong
```

安装依赖

```bash
#安装依赖会报错
[root@node1.local ~]# apt-get install fence-agents xorriso python3-gunicorn python3-pymongo
eading package lists... Done
Building dependency tree... Done
Reading state information... Done
You might want to run 'apt --fix-broken install' to correct these.
The following packages have unmet dependencies:
 cobbler : Depends: fence-agents but it is not going to be installed
           Depends: xorriso but it is not going to be installed
           Depends: python3-gunicorn but it is not going to be installed
 python3-pymongo : Depends: python3-bson (= 3.11.0-1ubuntu0.24.04.1) but it is not going to be installed
                   Recommends: python3-gridfs (>= 3.11.0-1ubuntu0.24.04.1) but it is not going to be installed
                   Recommends: python3-pymongo-ext but it is not going to be installed
E: Unmet dependencies. Try 'apt --fix-broken install' with no packages (or specify a solution).


#修复破损的依赖关系
[root@node1.local ~]# apt --fix-broken install

#安装成功
[root@node1.local ~]# apt-get install fence-agents xorriso python3-gunicorn python3-pymongo
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
fence-agents is already the newest version (4.12.1-2~exp1ubuntu4).
fence-agents set to manually installed.
xorriso is already the newest version (1:1.5.6-1.1ubuntu3).
xorriso set to manually installed.
python3-gunicorn is already the newest version (20.1.0-6).
python3-gunicorn set to manually installed.
python3-pymongo is already the newest version (3.11.0-1ubuntu0.24.04.1).
python3-pymongo set to manually installed.
0 upgraded, 0 newly installed, 0 to remove and 19 not upgraded.
```

安装cobbler

```bash
[root@node1.local cobbler]# dpkg -i /root/cobbler_3.4.0_all.deb
(Reading database ... 196119 files and directories currently installed.)
Preparing to unpack /root/cobbler_3.4.0_all.deb ...
Unpacking cobbler (3.4.0) over (3.4.0) ...
Setting up cobbler (3.4.0) ...
Processing triggers for man-db (2.12.0-4build2) ...

```

启动cobbler

```bash
mv /etc/cobbler/cobblerd.service /etc/systemd/system/

systemctl daemon-reload
systemctl start cobblerd
systemctl enable cobblerd
 
cobblerd check
```



## 三、centos安装

参考地址：https://blog.swireb.cn/archives/docs-011

准备工作

```bash
#关闭防火墙和selinux
systemctl disable firewalld.service
systemctl stop firewalld.service
sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config

#关闭了selinux需要重启服务器生效
reboot
```

安装 EPEL 仓库

```bash
yum install -y epel-release
```

更新仓库安装

```bash
yum update
yum install -y dhcp tftp-server xinetd debmirror pykickstart cobbler cobbler-web  

#组件作用简介
cobbler     #基础组件
cobbler-web #web组件
debmirror   #镜像管理工具
pykickstart #检查cobbler配置文件语法
httpd       #发布镜像
syslinux    #配置引导文件（生成pxelinux.0）
tftp-server #为PXE的客户端提供引导文件
dhcp        #为PXE的客户端提供IP地址、告知tftp的服务地址
```

Cobbler目录文件简介

```bash
rpm -ql cobbler
/etc/cobbler                  #配置文件目录
/etc/cobbler/settings         #cobbler主配置文件
/etc/cobbler/dhcp.template    #dhcp服务的配置模板
/etc/cobbler/tftpd.template   #tftp服务的配置模板
/etc/cobbler/rsync.template   #rsync服务的配置模板
/etc/cobbler/iso              #iso模板配置文件目录
/etc/cobbler/pxe              #pxe模板文件目录
/etc/cobbler/power            #电源的配置文件目录
/etc/cobbler/users.conf       #web服务授权配置文件
/etc/cobbler/users.digest     #用于web访问的用户名密码配置文件
/etc/cobbler/dnsmasq.template #dns服务的配置模板
/etc/cobbler/modules.conf     #cobbler模块配置文件
/var/lib/cobbler              #cobbler数据目录
/var/lib/cobbler/config       #配置文件
/var/lib/cobbler/kickstarts   #默认存放kickstart文件
/var/lib/cobbler/loaders      #存放的各种引导程序
/var/www/cobbler              #系统安装镜像目录
/var/www/cobbler/ks_mirror    #导入的系统镜像列表
/var/www/cobbler/images       #导入的系统镜像启动文件
/var/www/cobbler/repo_mirror  #YUM源存储目录
/var/log/cobbler              #日志目录
/var/log/cobbler/install.log  #客户端系统安装日志
/var/log/cobbler/cobbler.log  #cobbler日志 
```

Cobbler主配置文件修改

```bash
#生成密文密码
openssl passwd -1

#设置root密码
sed -i 's|^default_password_crypted.*|default_password_crypted: "$1$Nrt/tXCR$BrRthh4tFphGyCunrGWzi/"|g' /etc/cobbler/settings

#设置指定tftp服务IP地址
sed -i 's|^next_server.*|next_server: 192.1.1.211|g' /etc/cobbler/settings

#设置cobbler服务地址
sed -i 's|^server.*|server: 192.1.1.211|g' /etc/cobbler/settings

#cobbler接管dhcp（0为关闭 1为开启）
sed -i 's|^manage_dhcp.*|manage_dhcp: 0|g' /etc/cobbler/settings

#cobbler接管tftp（0为关闭 1为开启）
sed -i 's|^manage_tftpd.*|manage_tftpd: 1|g' /etc/cobbler/settings
      
#cobbler启动服务
systemctl enable --now httpd.service
systemctl enable --now cobblerd.service
```

Cobbler首次检查

```bash
cobbler check
1 : change 'disable' to 'no' in /etc/xinetd.d/tftp
2 : Some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements. #可以忽略（确保系统已经安装pxelinux）
3 : enable and start rsyncd.service with systemctl
4 : comment out 'dists' on /etc/debmirror.conf for proper debian support
5 : comment out 'arches' on /etc/debmirror.conf for proper debian support
6 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them #可以忽略
```

解决Cobbler检查报错

```bash
#报错1问题解决
cat << EOF | tee /etc/xinetd.d/tftp
service tftp
{
        socket_type             = dgram
        protocol                = udp
        wait                    = yes 
        user                    = root
        server                  = /usr/sbin/in.tftpd
        server_args             = -s /var/lib/tftpboot
        disable                 = no
        per_source              = 11
        cps                     = 100 2
        flags                   = IPv4
}
EOF

#报错3问题解决
systemctl enable --now rsyncd.service

#报错4、5问题解决
sed -i 's|@dists=.*|# @dists=|' /etc/debmirror.conf 
sed -i 's|@arches=.*|# @arches=|' /etc/debmirror.conf
```

Cobbler首次同步

```bash
#再次运行检查
cobbler check
1 : Some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.

#cobbler首次同步
cobbler sync
```

## **四、配置dhcp服务**

Cobbler接管dhcp

```bash
vim /etc/cobbler/dhcp.template  
subnet 10.99.88.0 netmask 255.255.255.0 {
     # option routers             10.99.88.55;
     # option domain-name-servers 127.0.0.1;
     option subnet-mask         255.255.255.0;
     range dynamic-bootp        10.99.88.100 10.99.88.254;
     default-lease-time         21600;
     max-lease-time             43200;
     next-server                $next_server;
     class "pxeclients" {
          match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
          if option pxe-system-type = 00:02 {
                  filename "ia64/elilo.efi";
          } else if option pxe-system-type = 00:06 {
                  filename "grub/grub-x86.efi";
          } else if option pxe-system-type = 00:07 {
                  filename "grub/grub-x86_64.efi";
          } else if option pxe-system-type = 00:09 {
                  filename "grub/grub-x86_64.efi";
          } else {
                  filename "pxelinux.0";
          }
     }

}
    
#cobbler主配置文件开启dhcp接管
sed -i 's|^manage_dhcp.*|manage_dhcp: 1|g' /etc/cobbler/settings 

#重新同步
systemctl restart cobblerd.service
cobbler sync  

#启动dhcp服务
systemctl enable --now dhcpd.service 
systemctl restart dhcpd.service
```

使用现有dhcp服务器-->定义了上面的模板下面会自动获取

```bash
#修改dhcp配置文件
vim /etc/dhcp/dhcpd.conf 
subnet 10.99.88.0 netmask 255.255.255.0 {
     # option routers             10.99.88.55;
     # option domain-name-servers 127.0.0.1;
     option subnet-mask         255.255.255.0;
     range dynamic-bootp        10.99.88.100 10.99.88.254;
     default-lease-time         21600;
     max-lease-time             43200;
     next-server                192.1.1.211;
     class "pxeclients" {
          match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
          if option pxe-system-type = 00:02 {
                  filename "ia64/elilo.efi";
          } else if option pxe-system-type = 00:06 {
                  filename "grub/grub-x86.efi";
          } else if option pxe-system-type = 00:07 {
                  filename "grub/grub-x86_64.efi";
          } else if option pxe-system-type = 00:09 {
                  filename "grub/grub-x86_64.efi";
          } else {
                  filename "pxelinux.0";
          }
     }

}
#启动dhcp服务
systemctl enable --now dhcpd.service 
systemctl restart dhcpd.service
```

## **五、其他相关服务配置**

配置tftp服务

```bash
#确保tftp的站点目录存在引导文件（cobbler检查问题的过程中已经修了tftp的配置文件）
ll /var/lib/tftpboot/
drwxr-xr-x  3 root root   4096 Mar  1 23:54 boot
drwxr-xr-x. 2 root root   4096 Oct 15  2019 etc
drwxr-xr-x. 2 root root   4096 Mar  1 23:54 grub  #UEFI启动菜单目录
drwxr-xr-x. 7 root root   4096 Mar  1 23:54 images
drwxr-xr-x. 2 root root   4096 Oct 15  2019 images2
-rw-r--r--. 2 root root  26140 Oct 31  2018 memdisk
-rw-r--r--. 2 root root  54964 Mar  1 23:54 menu.c32
drwxr-xr-x. 2 root root   4096 Oct 15  2019 ppc
-rw-r--r--. 2 root root  16794 Mar  1 23:54 pxelinux.0
drwxr-xr-x. 2 root root   4096 Mar  1 23:56 pxelinux.cfg #BIOS启动菜单目录
drwxr-xr-x. 2 root root   4096 Mar  1 23:54 s390x
-rw-r--r--  2 root root 198236 Feb  8 15:17 yaboot

#启动tftp服务
systemctl enable --now tftp.service  
systemctl enable --now xinetd.service
systemctl restart tftp.service 
systemctl restart xinetd.service
```

## 六、配置 Cobbler Server

参考地址：https://anjia0532.github.io/2019/02/22/cobbler-win10-win-server-2019/

#### 导入 Cobbler

使用 WinScp 等工具，将 winpe_win10_amd64.iso 上传到 Cobbler 服务器上

```bash
[root@localhost ~]# cobbler distro add --name=windows_10_x64 --kernel=/var/lib/tftpboot/memdisk --initrd=/root/winpe_win10_amd64.iso --kopts="raw iso"
[root@localhost ~]# touch /var/lib/cobbler/kickstarts/winpe.xml
[root@localhost ~]# cobbler profile add --name=windows_10_x64 --distro=windows_10_x64 --kickstart=/var/lib/cobbler/kickstarts/winpe.xml
```

#### 创建自动应答文件

```bash
[root@localhost kickstarts]# pwd
/var/lib/cobbler/kickstarts

[root@localhost kickstarts]# vim winpe.xml
<!--*************************************************
Windows 10 Answer File Generator
Created using Windows AFG found at:
;http://www.windowsafg.com

Installation Notes
Location: zh-CN
Notes: Enter your comments here...
**************************************************-->
<?xml version="1.0" encoding="utf-8"?>
<unattend
    xmlns="urn:schemas-microsoft-com:unattend">
    <settings pass="windowsPE">
        <component name="Microsoft-Windows-International-Core-WinPE" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <SetupUILanguage>
                <UILanguage>en-US</UILanguage>
            </SetupUILanguage>
            <InputLocale>0804:{81D4E9C9-1D3B-41BC-9E6C-4B40BF79E35E}{FA550B04-5AD7-411f-A5AC-CA038EC515D7}</InputLocale>
            <SystemLocale>zh-CN</SystemLocale>
            <UILanguage>zh-CN</UILanguage>
            <UILanguageFallback>zh-CN</UILanguageFallback>
            <UserLocale>zh-CN</UserLocale>
        </component>
        <component name="Microsoft-Windows-International-Core-WinPE" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <SetupUILanguage>
                <UILanguage>en-US</UILanguage>
            </SetupUILanguage>
            <InputLocale>0804:{81D4E9C9-1D3B-41BC-9E6C-4B40BF79E35E}{FA550B04-5AD7-411f-A5AC-CA038EC515D7}</InputLocale>
            <SystemLocale>zh-CN</SystemLocale>
            <UILanguage>zh-CN</UILanguage>
            <UILanguageFallback>zh-CN</UILanguageFallback>
            <UserLocale>zh-CN</UserLocale>
        </component>
        <component name="Microsoft-Windows-Setup" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <DiskConfiguration>
                <Disk wcm:action="add">
                    <CreatePartitions>
                        <CreatePartition wcm:action="add">
                            <Order>1</Order>
                            <Type>Primary</Type>
                            <Size>100</Size>
                        </CreatePartition>
                        <CreatePartition wcm:action="add">
                            <Extend>true</Extend>
                            <Order>2</Order>
                            <Type>Primary</Type>
                        </CreatePartition>
                    </CreatePartitions>
                    <ModifyPartitions>
                        <ModifyPartition wcm:action="add">
                            <Active>true</Active>
                            <Format>NTFS</Format>
                            <Label>System Reserved</Label>
                            <Order>1</Order>
                            <PartitionID>1</PartitionID>
                            <TypeID>0x27</TypeID>
                        </ModifyPartition>
                        <ModifyPartition wcm:action="add">
                            <Active>true</Active>
                            <Format>NTFS</Format>
                            <Label>OS</Label>
                            <Letter>C</Letter>
                            <Order>2</Order>
                            <PartitionID>2</PartitionID>
                        </ModifyPartition>
                    </ModifyPartitions>
                    <DiskID>0</DiskID>
                    <WillWipeDisk>true</WillWipeDisk>
                </Disk>
            </DiskConfiguration>
            <ImageInstall>
                <OSImage>
                    <InstallTo>
                        <DiskID>0</DiskID>
                        <PartitionID>2</PartitionID>
                    </InstallTo>
                    <InstallToAvailablePartition>false</InstallToAvailablePartition>
                </OSImage>
            </ImageInstall>
            <UserData>
                <AcceptEula>true</AcceptEula>
                <FullName>AnJia</FullName>
                <Organization>AnJia</Organization>
                <ProductKey>
                    <Key>VK7JG-NPHTM-C97JM-9MPGT-3V66T</Key>
                </ProductKey>
            </UserData>
        </component>
        <component name="Microsoft-Windows-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <DiskConfiguration>
                <Disk wcm:action="add">
                    <CreatePartitions>
                        <CreatePartition wcm:action="add">
                            <Order>1</Order>
                            <Type>Primary</Type>
                            <Size>100</Size>
                        </CreatePartition>
                        <CreatePartition wcm:action="add">
                            <Extend>true</Extend>
                            <Order>2</Order>
                            <Type>Primary</Type>
                        </CreatePartition>
                    </CreatePartitions>
                    <ModifyPartitions>
                        <ModifyPartition wcm:action="add">
                            <Active>true</Active>
                            <Format>NTFS</Format>
                            <Label>System Reserved</Label>
                            <Order>1</Order>
                            <PartitionID>1</PartitionID>
                            <TypeID>0x27</TypeID>
                        </ModifyPartition>
                        <ModifyPartition wcm:action="add">
                            <Active>true</Active>
                            <Format>NTFS</Format>
                            <Label>OS</Label>
                            <Letter>C</Letter>
                            <Order>2</Order>
                            <PartitionID>2</PartitionID>
                        </ModifyPartition>
                    </ModifyPartitions>
                    <DiskID>0</DiskID>
                    <WillWipeDisk>true</WillWipeDisk>
                </Disk>
            </DiskConfiguration>
            <ImageInstall>
                <OSImage>
                    <InstallTo>
                        <DiskID>0</DiskID>
                        <PartitionID>2</PartitionID>
                    </InstallTo>
                    <InstallToAvailablePartition>false</InstallToAvailablePartition>
                </OSImage>
            </ImageInstall>
            <UserData>
                <AcceptEula>true</AcceptEula>
                <FullName>AnJia</FullName>
                <Organization>AnJia</Organization>
                <ProductKey>
                    <Key>VK7JG-NPHTM-C97JM-9MPGT-3V66T</Key>
                </ProductKey>
            </UserData>
        </component>
    </settings>
    <settings pass="offlineServicing">
        <component name="Microsoft-Windows-LUA-Settings" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <EnableLUA>false</EnableLUA>
        </component>
    </settings>
    <settings pass="offlineServicing">
        <component name="Microsoft-Windows-LUA-Settings" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <EnableLUA>false</EnableLUA>
        </component>
    </settings>
    <settings pass="generalize">
        <component name="Microsoft-Windows-Security-SPP" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <SkipRearm>1</SkipRearm>
        </component>
    </settings>
    <settings pass="generalize">
        <component name="Microsoft-Windows-Security-SPP" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <SkipRearm>1</SkipRearm>
        </component>
    </settings>
    <settings pass="specialize">
        <component name="Microsoft-Windows-International-Core" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <InputLocale>0804:{81D4E9C9-1D3B-41BC-9E6C-4B40BF79E35E}{FA550B04-5AD7-411f-A5AC-CA038EC515D7}</InputLocale>
            <SystemLocale>zh-CN</SystemLocale>
            <UILanguage>zh-CN</UILanguage>
            <UILanguageFallback>zh-CN</UILanguageFallback>
            <UserLocale>zh-CN</UserLocale>
        </component>
        <component name="Microsoft-Windows-International-Core" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <InputLocale>0804:{81D4E9C9-1D3B-41BC-9E6C-4B40BF79E35E}{FA550B04-5AD7-411f-A5AC-CA038EC515D7}</InputLocale>
            <SystemLocale>zh-CN</SystemLocale>
            <UILanguage>zh-CN</UILanguage>
            <UILanguageFallback>zh-CN</UILanguageFallback>
            <UserLocale>zh-CN</UserLocale>
        </component>
        <component name="Microsoft-Windows-Security-SPP-UX" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <SkipAutoActivation>true</SkipAutoActivation>
        </component>
        <component name="Microsoft-Windows-Security-SPP-UX" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <SkipAutoActivation>true</SkipAutoActivation>
        </component>
        <component name="Microsoft-Windows-SQMApi" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <CEIPEnabled>0</CEIPEnabled>
        </component>
        <component name="Microsoft-Windows-SQMApi" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <CEIPEnabled>0</CEIPEnabled>
        </component>
        <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <ComputerName>AnJia-PC</ComputerName>
            <ProductKey>VK7JG-NPHTM-C97JM-9MPGT-3V66T</ProductKey>
        </component>
        <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <ComputerName>AnJia-PC</ComputerName>
            <ProductKey>VK7JG-NPHTM-C97JM-9MPGT-3V66T</ProductKey>
        </component>
    </settings>
    <settings pass="oobeSystem">
        <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="x86" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <AutoLogon>
                <Password>
                    <Value></Value>
                    <PlainText>true</PlainText>
                </Password>
                <Enabled>true</Enabled>
                <Username>AnJia</Username>
            </AutoLogon>
            <OOBE>
                <HideEULAPage>true</HideEULAPage>
                <HideOEMRegistrationScreen>true</HideOEMRegistrationScreen>
                <HideOnlineAccountScreens>true</HideOnlineAccountScreens>
                <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
                <NetworkLocation>Work</NetworkLocation>
                <SkipUserOOBE>true</SkipUserOOBE>
                <SkipMachineOOBE>true</SkipMachineOOBE>
                <ProtectYourPC>1</ProtectYourPC>
            </OOBE>
            <UserAccounts>
                <LocalAccounts>
                    <LocalAccount wcm:action="add">
                        <Password>
                            <Value></Value>
                            <PlainText>true</PlainText>
                        </Password>
                        <Description>AnJia</Description>
                        <DisplayName>AnJia</DisplayName>
                        <Group>Administrators</Group>
                        <Name>AnJia</Name>
                    </LocalAccount>
                </LocalAccounts>
            </UserAccounts>
            <RegisteredOrganization>AnJia</RegisteredOrganization>
            <RegisteredOwner>AnJia</RegisteredOwner>
            <DisableAutoDaylightTimeSet>false</DisableAutoDaylightTimeSet>
            <FirstLogonCommands>
                <SynchronousCommand wcm:action="add">
                    <Description>Control Panel View</Description>
                    <Order>1</Order>
                    <CommandLine>reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\ControlPanel" /v StartupPage /t REG_DWORD /d 1 /f</CommandLine>
                    <RequiresUserInput>true</RequiresUserInput>
                </SynchronousCommand>
                <SynchronousCommand wcm:action="add">
                    <Order>2</Order>
                    <Description>Control Panel Icon Size</Description>
                    <RequiresUserInput>false</RequiresUserInput>
                    <CommandLine>reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\ControlPanel" /v AllItemsIconView /t REG_DWORD /d 0 /f</CommandLine>
                </SynchronousCommand>
                <SynchronousCommand wcm:action="add">
                    <Order>3</Order>
                    <RequiresUserInput>false</RequiresUserInput>
                    <CommandLine>cmd /C wmic useraccount where name="AnJia" set PasswordExpires=false</CommandLine>
                    <Description>Password Never Expires</Description>
                </SynchronousCommand>
            </FirstLogonCommands>
            <TimeZone>China Standard Time</TimeZone>
        </component>
        <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS"
            xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
            <AutoLogon>
                <Password>
                    <Value></Value>
                    <PlainText>true</PlainText>
                </Password>
                <Enabled>true</Enabled>
                <Username>AnJia</Username>
            </AutoLogon>
            <OOBE>
                <HideEULAPage>true</HideEULAPage>
                <HideOEMRegistrationScreen>true</HideOEMRegistrationScreen>
                <HideOnlineAccountScreens>true</HideOnlineAccountScreens>
                <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
                <NetworkLocation>Work</NetworkLocation>
                <SkipUserOOBE>true</SkipUserOOBE>
                <SkipMachineOOBE>true</SkipMachineOOBE>
                <ProtectYourPC>1</ProtectYourPC>
            </OOBE>
            <UserAccounts>
                <LocalAccounts>
                    <LocalAccount wcm:action="add">
                        <Password>
                            <Value></Value>
                            <PlainText>true</PlainText>
                        </Password>
                        <Description>AnJia</Description>
                        <DisplayName>AnJia</DisplayName>
                        <Group>Administrators</Group>
                        <Name>AnJia</Name>
                    </LocalAccount>
                </LocalAccounts>
            </UserAccounts>
            <RegisteredOrganization>AnJia</RegisteredOrganization>
            <RegisteredOwner>AnJia</RegisteredOwner>
            <DisableAutoDaylightTimeSet>false</DisableAutoDaylightTimeSet>
            <FirstLogonCommands>
                <SynchronousCommand wcm:action="add">
                    <Description>Control Panel View</Description>
                    <Order>1</Order>
                    <CommandLine>reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\ControlPanel" /v StartupPage /t REG_DWORD /d 1 /f</CommandLine>
                    <RequiresUserInput>true</RequiresUserInput>
                </SynchronousCommand>
                <SynchronousCommand wcm:action="add">
                    <Order>2</Order>
                    <Description>Control Panel Icon Size</Description>
                    <RequiresUserInput>false</RequiresUserInput>
                    <CommandLine>reg add "HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\ControlPanel" /v AllItemsIconView /t REG_DWORD /d 0 /f</CommandLine>
                </SynchronousCommand>
                <SynchronousCommand wcm:action="add">
                    <Order>3</Order>
                    <RequiresUserInput>false</RequiresUserInput>
                    <CommandLine>cmd /C wmic useraccount where name="AnJia" set PasswordExpires=false</CommandLine>
                    <Description>Password Never Expires</Description>
                </SynchronousCommand>
            </FirstLogonCommands>
            <TimeZone>China Standard Time</TimeZone>
        </component>
    </settings>
</unattend>
```

## 七、配置 samba

#### 安装 samba

```bash
yum install samba -y
```

#### 修改 smb config

```bash
[root@localhost ~]# vi /etc/samba/smb.conf

# /etc/samba/smb.conf
[global]
log file = /var/log/samba/log.%m
max log size = 5000
security = user
guest account = nobody
map to guest = Bad User
load printers = yes
cups options = raw

[share]
comment = share directory目录
path = /smb/
directory mask = 0755
create mask = 0755
guest ok=yes
writable=yes
```

#### 启动 smb 服务

```bash
[root@localhost ~]# service smb start
[root@localhost ~]# systemctl enable smb

```

#### 挂载 win10 系统

通过 winscp 等软件将 cn_windows_10_business_edition_version_1809_updated_sept_2018_x64_dvd_84ac403f.iso 上传到 cobbler 服务器上,并将创建的应答文件，上传到 cobbler `/smb/win/win10_x64_bios_auto.xml`

```bash
[root@localhost ~]# mkdir -p /smb/win
[root@localhost ~]# mount -o loop,ro zh-cn_windows_10_enterprise_ltsc_2021_x64_dvd_033b7312.iso /mnt/
[root@localhost ~]# cp -r /mnt/* /smb/win
[root@localhost ~]# umount /mnt/
```

## 八、装 Windows10

从 vmware 创建一台内存 4G，cpu2 核，磁盘 60G 的空盘，win10 虚拟机，然后开机。记得选 BIOS，别选 UEFI。
![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/6e56c329fdea4466add490819b39036e.png)

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/3b43708c6eab44819bbcb57010aa3f81.png)

![在这里插入图片描述](https://gitee.com/ljh00928/csdn/raw/master/img/4516113a6b4347578738dae115e4a254.png)