---
layout: post
title: Ubuntu14.04 使用 Openswan 搭建 IPSec VPN
categories:[blog]
tags:[Tech,Teach,Guide,VPN]
description:
---
#Ubuntu14.04 使用 Openswan 搭建 IPSEC VPN
 

Ubuntu使用14.04版本。 我尝试了一下Ubuntu16.04的版本，本教程对16.04不适用，16.04版本有些地方改动较大。

搭虚拟机的软件使用VMware，官网下载，百度搜索“VMware注册机”即可找到序列号破解。

正文
---
---

##配置虚拟网络

首先介绍一下我们第一步的目的：创建两台主机、两台服务器

　　left<---->leftserver<---->rightserver<---->right

然后通过配置ip和编辑虚拟网络，让left和right能ping通。

###1.编辑虚拟网络

　　首先创建虚拟机的时候怎么选择网络的模式呢？

　　　**桥接模式**：代表虚拟机在主机所在的子网中分配一个IP地址，相当于主机所在的子网里的另一台主机

　　　**NAT模式**：将主机和虚拟机接在一个NAT的路由下。

　　　**仅主机模式**：虚拟机是一台完全隔离的新主机。要上外网则必须经过主机。

　　我们搭建一个纯内部通信的虚拟网络，为每个网卡选择仅主机模式。但是默认的仅主机模式只有一个网段，而我们希望使用多个原本不相通的网段，然后通过配置来使得left和right能ping通。如下：

　　　　left<--(VMnet2)-->leftserver<--(VMnet3)-->rightserver<--(VMnet4)-->right

　　再加上网卡，最终的效果如下：（主机一张网卡，服务器两张网卡，添加方式在虚拟机的设置里）

　　　　left{eth0}<--(VMnet2)-->{eth1}leftserver{eth0}<--(VMnet3)-->{eth0}rightserver{eth1}<--(VMnet4)-->{eth0}right

　　![topology](http://home.ustc.edu.cn/~tzh0799/picture/1.png)
　　
　　可以把每个VMnet想象为一个路由器。

　　记得要先在{编辑->虚拟网络编辑器}中将VMnet2，VMnet3，VMnet4添加到网络中。并设置为仅主机模式，但是要去掉DHCP模式。（因为一会儿我们配置静态IP）

　　添加完之后如下：

　　![topology](http://home.ustc.edu.cn/~tzh0799/picture/5.png)

　　在VMware中创建虚拟机->典型配置->安装程序光盘映像iso->自己起名设密码，直到出现“自定义硬件”时在网络适配器中按照图中所示选择VMnet。

###2.配置静态IP

　　编辑文件：
```
sudo vim /etc/network/interfaces
```
　　以left-server为例：（对每个主机都进行配置，配置完重启，注意eth0和1的编号与你添加的网络适配器顺序要对应）
```
　　auto eth0
　　iface eth0 inet static
　　address 192.168.229.128
　　gateway 192.168.229.129
　　netmask 255.255.255.0
　　auto eth1
　　iface eth1 inet static
　　address 192.168.244.129
　　network 192.168.244.0
　　netmask 255.255.255.0
```
　　然后编辑两个服务器的转发功能文件：（ubuntu默认不打开转发功能，要想把主机改为网关，必须打开转发功能，目的是让left-server的eth0和eth1连通）
```bash
sudo vim /etc/sysctl.conf
```
　　将文件中net.ipv4.ip_forward=1前的#去掉（#是注释的意思）

　　然后运行：
```
sudo sysctl -p
```
　　使改动生效。

　　重启之后，如果以上过程做好了，那left和right主机之间是可以ping通的。如果不通，检查/etc/network/interfaces文件是否细节错误，再检查服务器转发功能是否打开，再检查eth0、1编号设置的静态ip是否对应VMnet对应的网段。

　　最好使用ping xxx -R的命令，查看显示的路由路径是否正确
　　
　　![topology](http://home.ustc.edu.cn/~tzh0799/picture/2.png)
　　

##搭建IPsec VPN

 

 　　　在两台服务器上安装openswan：(注意安装过程中不要安装X.509证书，会有提示)
```linux
sudo apt-get install openswan
```
　　　　修改配置文件：（此处是修改随机数发生器，这一步是必要的）
```
sudo vim /usr/lib/ipsec/newhostkey
```
　　　　修改第64行，改为：
```
ipsec rsasigkey $verbose --random /dev/urandom $host $bits
```
　　　　然后运行以下命令生成公钥：
```
sudo ipsec newhostkey --output /etc/ipsec.conf
```
　　　　用以下命令可以查看本服务器的公钥（left或者right都可以）
```
sudo ipsec showhostkey --left
```
![topology](http://home.ustc.edu.cn/~tzh0799/picture/3.png)
　　　
　　　　以上步骤对两个服务器都要做一遍。

　　　　然后接下来是最主要的步骤，编辑配置文件:
```
sudo vim /etc/ipsec.conf
```
　　　　在config setup部分，将prostack改为netkey，即prostack=netkey

　　　　然后修改conn sample如下：
　　　　![topology](http://home.ustc.edu.cn/~tzh0799/picture/4.png)

　　　　注意：

　　　　　　　　left = 左侧服务器网关的外部ip

　　　　　　　　leftsubnet = 左侧子网的network / netmask

　　　　　　　　leftnexthop = %defaultroute 会自动将本地的缺省路由用interfaces里的指定路由填充

　　　　　　　　leftid = @left

　　　　　　　　leftrsasigkey = 用上边查看公钥命令看到的公钥，复制下来粘贴在这里

　　　　　　　　右服务器也是这样操作，注意公钥是不同的。

　　　　然后检验一下各个条件是否已经满足：
```
sudo ipsec verify
```
　　![verify](http://home.ustc.edu.cn/~tzh0799/picture/6.png)

　　一开始你的右边很可能有很多难看的**[FAILED]**

　　以下是解决方案：（每个FAILED对应不同的解决方案，自行往下边找）

　　**1. checking for IPsec support in kernel [FAILED]**

　　　　　　whackµPluto is not running(no "/var/run/pluto/pluto.ctl") [FAILED]

　　　　　　解决方案：　运行
```
sudo ipsec setup start
```
　　**2. Hardware RNG detected, testing if used properly [FAILED]**

　　　　　　解决方案：   运行
```
sudo apt-get install rng-tools
```
　　**3. Checking Ip forwarding [FAILED]**

　　　　　　解决方案：　　
　　　　　　检查转发功能是否打开（IP_forward是否=1），但是即使=1，此项的failed依然可能显示，不必在意。

　　　4. **NETKEY detected, testing for disabled ICMP send redirects [FAILED]
　　　　　　Please disable /proc/sys/net/ipv4/conf/*/send redirects
　　　　　　or NETKEY will cause the sending of bogus ICMP redirects!
　　　　 NETKEY detected, testing for disabled ICMP accept redirects [FAILED]
　　　　　　Please disable /proc/sys/net/ipv4/conf/*/accept redirects
　　　　　　or NETKEY will accept bogus ICMP redirects!**

　　　　　　解决方案：

　　　　　　创建脚本 a.sh
```
sudo vim a.sh
```
将以下内容添加到 a.sh中，目的是关闭ICMP重定向
```
#! /bin/bash
# Disable send redirects
echo 0 > /proc/sys/net/ipv4/conf/all/send redirects
echo 0 > /proc/sys/net/ipv4/conf/default/send redirects
echo 0 > /proc/sys/net/ipv4/conf/eth0/send redirects
echo 0 > /proc/sys/net/ipv4/conf/eth1/send redirects
echo 0 > /proc/sys/net/ipv4/conf/lo/send redirects
# Disable accept redirects
echo 0 > /proc/sys/net/ipv4/conf/all/accept redirects
echo 0 > /proc/sys/net/ipv4/conf/default/accept redirects
echo 0 > /proc/sys/net/ipv4/conf/eth0/accept redirects
echo 0 > /proc/sys/net/ipv4/conf/eth1/accept redirects
echo 0 > /proc/sys/net/ipv4/conf/lo/accept redirects
echo 1 > /proc/sys/net/core/xfrm larval drop
```
　　　　然后运行：
```
sudo chmod a+x a.sh
```
　　　　然后执行：
```
sudo bash a.sh
```
　　　　此时可以再回去进行一次检验
```
sudo ipsec verify
```
　　　　注意：

　　　　a.sh脚本每次重启都要运行一次，因为每次关机都会失效。

##建立连接

```
sudo ipsec auto --up sample
```

![sample](http://home.ustc.edu.cn/~tzh0799/picture/7.png)
　　　　
　　　　上图表示建立连接成功。此时再在left主机上ping通right主机试一下。若出错，只需要检查上述步骤

##抓包分析

　　要自己下载tcpcump用来监听和抓包（系统一般自带）和wireshark（自己下载）进行分析
```
sudo apt-get install tcpdump
sudo apt-get install wireshark
```
　　在一台服务器上，运行以下命令打开tcpdump开始监听端口：
```
sudo tcpdump -i eth0
```
　　然后在left主机上ping通right主机，抓包如下：

  ![tcpdump](http://home.ustc.edu.cn/~tzh0799/picture/8.png)

　　可以发现，已经是ESP格式的ipsec传输了，我们的目的达到了。

##Addition：

　　若要再用wireshark分析

　　　　首先使用以下命令用tcpdump抓100个包，存放到/home/data.pcap文件中，然后用wireshark分析：（-s 0表示每个包都存下完整信息）
```
tcpdump -i eth0 -c 100 -s 0 -w /home/data.pcap
wireshark /home/data.pcap
```
　　　　由于我们之前的配置文件设置中，auto=start，表示一建立连接立马开始密钥协商。所以我们抓包时已经密钥协商结束了。

　　　　要想抓到密钥协商时的包，步骤如下：

　　　　把一台服务器关机以leftserver为例，把leftserver关机，在rightserver中用tcpdump -i eth0 -c 100 -s 0 -w /home/data.pcap命令打开tcpdump监听，然后再把leftserver开机，便可以抓到。

　　　　然后运行wireshark /home/data.pcap

　　　　在Filter过滤中输入：ISAKMP 。结果如下：

![ISAKMP](http://home.ustc.edu.cn/~tzh0799/picture/9.png)

##Future

具体抓包分析见下一篇博文[IPSEC IKE协商过程抓包分析](http://ZacharyTan.top)

 


