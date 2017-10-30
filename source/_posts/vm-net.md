title: 虚拟机及docker的网络连接
category: [cloud]
tags:
date: 2016-09-20
---
网络是云计算中重要的基础设施。这里比较了VirtualBox（简写为vbox），VMware，Hyper-V，KVM这些虚拟机管理器（VMM）及docker的网络连接方式。


<!--more-->

---

<!-- TOC -->

    - [title: 虚拟机及docker的网络连接](#title-%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%8F%8Adocker%E7%9A%84%E7%BD%91%E7%BB%9C%E8%BF%9E%E6%8E%A5)
- [路由器，NAT，交换机](#%E8%B7%AF%E7%94%B1%E5%99%A8%EF%BC%8Cnat%EF%BC%8C%E4%BA%A4%E6%8D%A2%E6%9C%BA)
- [单机的网络地址转换（NAT）](#%E5%8D%95%E6%9C%BA%E7%9A%84%E7%BD%91%E7%BB%9C%E5%9C%B0%E5%9D%80%E8%BD%AC%E6%8D%A2%EF%BC%88nat%EF%BC%89)
- [NAT网络](#nat%E7%BD%91%E7%BB%9C)
- [Host-Only 仅主机](#host-only-%E4%BB%85%E4%B8%BB%E6%9C%BA)
- [桥接](#%E6%A1%A5%E6%8E%A5)
- [Docker的桥接模式](#docker%E7%9A%84%E6%A1%A5%E6%8E%A5%E6%A8%A1%E5%BC%8F)
- [内部网络/LAN区段](#%E5%86%85%E9%83%A8%E7%BD%91%E7%BB%9Clan%E5%8C%BA%E6%AE%B5)
- [选择哪种连接方式？](#%E9%80%89%E6%8B%A9%E5%93%AA%E7%A7%8D%E8%BF%9E%E6%8E%A5%E6%96%B9%E5%BC%8F%EF%BC%9F)
- [Bonus](#bonus)

<!-- /TOC -->

# 路由器，NAT，交换机

先简单介绍一下[NAT](https://zh.wikipedia.org/wiki/%E7%BD%91%E7%BB%9C%E5%9C%B0%E5%9D%80%E8%BD%AC%E6%8D%A2)。
家用的无线路由器就使用了NAT。连接到同一个无线路由器的手机，电脑等设备在使私有IP网段的局域网（LAN）中的多个设备经路由器的外网（WAN）IP访问外网。常见的家用无线路由器一般会使用192.168.0.0，192.168.1.0这样的IP网段。下面是从TP-LINK网站上找的一个图。路由器的WAN口接到了电信或者联通这些运营商提供的接口上，计算机可以用有线连接到路由器的LAN口，手机、平板也可以通过无线wifi信号连接到路由器。

接到路由器上面的设备组成了一个局域网（LAN），这些设备彼此直接可以通过私网IP直接通信（Windows主机之间还可以通过机器名来访问），它们连接外网时则共享WAN口的IP，从外部来看，所有的连接都来自同一个IP。如果访问[ip.cn](http://ip.cn)，显示的IP跟当前计算机的IP是不同的。

网络基础课上讲到IP v4有三类私有IP地址：

+ 10.0.0.0 ~ 10.255.255.255      （A类）；
+ 172.16.0.0 ~ 172.31.255.255    （B类）；
+ 192.168.0.0 ~ 192.168.255.255  （C类）；


![家用无线路由器拓扑](/img/wifi-router.png)

接到运营商的路由器WAN口有一个IP地址，一般是一个Internet（公网）上的动态IP，这个WAN口的IP不是固定的，每次重新连接可能会变化。如果需要一个静态的公网IP，一般则需要向运营商申请。

在学校实验室的网络连接与此类似，每台机器都有一个公网的IP地址。访问[ip.cn](http://ip.cn) 或者在ubuntu上执行 `curl ip.cn` 可以查到这个公网IP，也可以执行`ip a` 、`ifconfig` 命令来查询。公网IP理论上是可以被外界访问到的，但学校出口的防火墙屏蔽了外部 **发起** 的访问，只有经授权的IP和端口才能被外部访问，这样就减少了被外部攻击的可能，也就不能随便搭网站了。
曾经家里使用中国移动的宽带，分配的WAN口是一个B类的私有IP地址，也就是相当于我家的路由器又连到了中国移动的路由器的局域网里了，这样外部是不能访问这个WAN口IP的。

>学校访问外网先要登陆“网络接入认证系统”，这个系统登陆后只允许登陆设备的IP访问外网，而且不允许多台设备（多个IP）同时登录。如果自己先用路由器接到学校的网络上，然后再把多台设备接到自己的路由器上，只要这些设备中有一台登陆了“网络接入认证系统”，其它设备也就可以访问外网了。因为从认证系统看来，这些设备的IP地址都是路由器WAN口的IP地址。

另外，路由器本身除了有WAN口IP之外，还有一个LAN口IP，一般是192.168.0.1/192.168.1.1，这样我们才可以通过这个IP登陆到路由器上修改相关设置。

那么问题来了：
1、路由器LAN的设备可以访问外网，这是没有问题的，但外网怎么访问私有IP的内部设备呢？
2、外网怎么区分路由器LAN的不同设备呢？

实际上外部网络什么也不用干，它看到的只是路由器的WAN口IP。这两个问题都是路由器通过NAT（网络地址转换）来解决的。内网向外网发送一个数据包时，路由器把数据包的源地址（实际是上还有端口号，即IP:Port格式，比如默认的http是80端口），也就是内网设备的私网IP改成自己的WAN口IP，并给不同的设备（随机）分配不同的端口号，并把随机端口号与内网设备的IP:Port的映射关系保存下来。路由器接收到外部返回的应答数据包后，根据端口号查询到实际的内网设备IP:Port，再改写数据包的目标地址，转发给LAN上的内网设备。因此NAT是分两个方向的，分别称为源NAT (SNAT) 和 目标NAT (DNAT)。

如果外网直接访问路由器的WAN口IP，路由器找不到端口映射关系，就会直接drop掉这个数据包，导致外网不能直接访问内部设备。NAT默认要求一个网络通信必须由内部设备发起。也可以在路由器中设置好固定的端口映射，这样路由器就知道该转发给哪个设备，当然它还是可以判断出来是内部设备主动发起的通信，还是由外部设备发起的。还有一种DMZ技术，让路由器直接把一台内网设备暴露给外网，一般总还是有一些空闲的端口的，所以DMZ不会影响其它内网设备的联网。

把路由器的WAN口那一块去掉，剩下的LAN口那部分功能就是一个交换机了，交换机所有的端口都是对等的，所连接的设备组成了一个LAN。
网络基础课上会介绍交换机是二层设备，路由器是三层设备。**所谓二层，即物理链路层，最常见的就是以太网，设备之间以MAC地址区分；三层是网络层，设备之间以IP地址区分。三层的数据是封装在二层之中的。交换机只看数据包的MAC首部，而路由器则只看IP首部。但要是交换机偷看了IP首部也不会爆炸，而是变成更高级的三层交换机来抢路由器的生意了。具体三层交换机跟路由器的差别，因水平有限就不谈了。** 网络编程基本都是在三层（IP）四层（TCP/UDP），很少直接接触二层的（MAC）。
另外，家用路由器一端是公网，另一端是不能直接路由的私网IP段，而数据中心或电信路由器各端口（不止两个）连接的一般都是公网，可以直接路由，不需要使用NAT。

<center>~</center>

下面的表格是几种虚拟机管理器（VMM）支持的网络连接方式，同一行的网络连接方式实现的功能是基本相同的，不过在不同的VMM里叫法有所不同。

![几种虚拟机的网络连接方式](/img/vnet.png)

>注：**NAT网络**，Host->VM的情况，VMware在Host创建了虚拟网卡，不需要端口映射，Host可以直接通过IP来访问VM。

# 单机的网络地址转换（NAT）
 只有vbox支持 **单机NAT** 方式。这种方式与下面的 **NAT网络** 的唯一区别是，不同的VM之间不能互相通信。这个特性是为了方便创建多个单机VM，而不会彼此干扰。
 虽然是针对单机的，实际上NAT也有多个网络设备，包括一个网关，IP是`10.0.2.2`，一个DHCP服务器，也是`10.0.2.2`，VM的IP一般是`10.0.2.15`。
 不管使用NAT的有多少个VM，它们的IP都是一样的。

 一个VM可以设置多个网卡，如果这些网卡有多个使用NAT模式，那么它们的IP区段就会递增，分别为`10.0.2.0/24`，`10.0.3.0/24`等。不过同一个VM设置多个NAT网卡并没有什么必要。

 VM可以直接访问Host，默认IP也是`10.0.2.2`，但这个IP在Host是看不到的，即Host跟VM不在同一个LAN，Host不能直接访问VM的IP。
 Host要想访问VM，需要设置端口映射，即设置`HostIP:HostPort`与`VM-IP:VM-Port`的关联，这样Host就可以通过`HostIP:HostPort`来访问VM的端口了。如果映射的HostIP是可以从外部路由的，那么外部也可以通过`HostIP:Host-Port`来访问VM的指定端口。
 可以添加多个映射规则来暴露不同的端口。

# NAT网络
 在VMware中则直接称为 **NAT** 。vbox刻意把这 **NAT** 和 **NAT网络** 这2种方式区分开，可能是因为他们认为不仅需要实现网络连通，还要能够实现隔离吧。
 vbox需要在 **全局配置->网络** 中增加一个 **NAT网络** 的虚拟网卡才能使用NAT网络，但这个网卡在Host的网络设备里是看不到的。添加的第一个NAT网络的网段是`10.0.2.0/24`，网关是`10.0.2.1`，DHCP服务器是`10.0.2.3`，Host是`10.0.2.2`，VM的IP是由DHCP自动分配的。
 连接到同一个NAT网络的VM组成了一个LAN。这种情况跟前面提到的家用路由器的连接是很相似的，连接到路由器的多个设备是在一个私有IP网段的LAN，路由器有一个外网的IP，各种设备可以通过路由器连接外网，不做端口映射的话，外网不能直接访问内网的设备。

 >外网是可以与内网通信的，否则我们就不能看到网站返回的网页了，但必须要内网设备发起连接，外网响应。这是因为内网的IP是私有的，在 **公网** 上是不可路由的。
 >这里 **外网** 是Host之外的网络，它可能是一个Internet IP（**公网**），也可能是某个公司内部的私有IP地址的网络。
 >如果可以修改 **外网** 的路由的话，还是可以实现不做端口映射，通过路由协议来路由到内网设备的，但一般只有运营商才有这样的能力。

 连接到不同的虚拟网卡的VM，即便IP网段是相同，也不能连通。
 vbox可以添加多个NAT网络，它们可以使用相同的默认IP网段`10.0.2.0/24`，这不会彼此产生冲突，当然也可将其网段改为其它地址，如`10.0.3.0/24`，或`192.168.2.0/24`这样的，但`192.168.x.x`网段的DHCP可能不能正常工作，需设置静态IP。

 VMware的NAT网络设置有所不同。VMware安装后，会在（Windows）Host添加一个VMnet8的虚拟网卡，工作在NAT模式。在VM中选择网卡为VMnet8和设置使用NAT模式的效果是一样的。有了这个虚拟网卡，Host也就在NAT网络的LAN里面了，所以VMware的NAT网络模式下Host是可以直接访问VM的，这点比vbox要方便些。

 VMware内置了VMnet0 ~ VMnet19共20个虚拟网卡可用，每个虚拟网卡对应了一个虚拟LAN，可以工作在 **NAT**、**仅主机** 和 **桥接** 三种不同的模式之一，但又 **限制只能有一个虚拟网卡工作在NAT模式** （这个限制是很奇怪的）。不过VMware在NAT模式下可以更改 **默认网关的IP**，及DHCP、DNS的设置。

>注意，vbox在两种NAT模式下都有一个坑：
>VM在使用DHCP分配IP时能正常访问外网，但如果在VM中设置静态IP，**即便这些值与DHCP分配到的值一模一样，也不能访问外网！**
>虽然不能访问外网，但内网能正常访问，说明可能是DNS设置的问题。
>
>在这个[博客](http://geekynotebook.orangeonthewall.com/configure-static-ip-on-nat-in-oracle-virtualbox/) 中介绍了同样的问题（博客里的DNS IP `10.0.2.3`和`/etc/resolve.conf`设置在ubuntu上不能工作）。
>对Ubuntu，需在 `/etc/network/interfaces` 设置网卡

```
auto eth0
iface eth0 inet static
address 10.0.2.15
netmask 255.255.255.0
gateway 10.0.2.2

dns-nameservers 10.0.2.1
```

>**设置静态IP后，VM还要执行下面2条命令才会使用Host的DNS！**
>vbox在DHCP模式下会自动使用Host的DNS，但设置静态IP后默认不再使用Host的DNS，导致VM无法连接外网。

```
VBoxManage modifyvm "VM-Name" --natdnsproxy1 on
VBoxManage modifyvm "VM-Name" --natdnshostresolver1 on
```

>其中`VBoxManage`是vbox的命令行管理工具，在vbox的安装目录下，默认位置是`C:\Program Files\Oracle\VirtualBox\`。
>
>vbox 5.0.0 版有这个坑，而5.0.2 及以后的版本，**NAT网络** 方式已经不需要执行上面2条命令了，但`/etc/network/interfaces`里dns-nameservers的设置还是需要的。
>另外，经过实验发现gateway设置为`10.0.2.1` 或 `10.0.2.2`都可以连网，但DNS必须是`10.0.2.1`。
>
>VMware没有这个坑。

# Host-Only 仅主机
 这种连接方式是4种VMM都支持的，它的使用很简单。
 Host-Only模式下，vbox、VMware和Hyper-V都会在Host添加一个虚拟网卡，使用同一个虚拟网卡的VM会连接到同一个虚拟LAN，而且Host也在这个LAN。Host，VM之间都可以方便的连通，不需要端口映射，但VM不能访问外网。
 当我们创建了多个VM，并把这些VM连接到一个虚拟LAN，这就算是一个小型的 **虚拟 数据中心** 了。一个现实的数据中心里，除了多台服务器，还有 **交换机**，集中式存储设备，比如iSCSI，SAN，NAS等。一个机架里的多台服务器连接到机架顶部的交换机（ToR），多个机架交换机再连接到核心交换机。
 实际上数据中心的网络结构是很复杂的，并不只有一个LAN，而是分成了外部网络、内部业务网络、管理网络、存储网络等多个网络，还会划分成多个子网。
 Host-Only连接方式物相当于机架顶部的交换机（ToR），所以Hyper-V的叫法：**内部交换机** 是很贴切的。

 还可以添加多个虚拟网卡，组成多个彼此隔离的LAN（Host分别有一个虚拟网卡挂在每个LAN中）。
 
 利用Windows的网络共享功能，将外部网络共享给Host的虚拟网卡，然后将VM设置为静态IP，网关和DNS设置为`192.168.137.1`，这样就可以连接外网了，但只能共享给一个虚拟网卡，而且IP网段必须是`192.168.137.0/24`。
 复杂点的办法是在Host配置NAT，以Hyper-V为例，它没有NAT网络，参考[Set up a NAT network for Hyper-V](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/setup-nat-network)文档，在Powershell下以管理员权限执行
```
New-VMSwitch -SwitchName "NAT" -SwitchType Internal
```

查看interfere index：`Get-NetAdapter`，即下面命令中虚拟网卡的`-InterfaceIndex 38`。
```
New-NetIPAddress -IPAddress 10.0.0.2 -PrefixLength 24 -InterfaceIndex 38
New-NetNat -Name NATx -InternalIPInterfaceAddressPrefix 10.0.0.0/24
```

这样就创建了一个可以通过Host以NAT方式访问外网的 **内部交换机**，不过这个内部交换机还缺少DHCP服务，需要为VM静态分配IP。

# 桥接
 桥接是最方便的一种连接方式。它需要绑定到物理网卡。如果Host的网络连接正常，那么VM一般也就没什么问题了，但如果Host的网络出现了故障，那么VM也无法联网了，即便是同一个Host的VM之间也不能正常通信。
 桥接模式下VM与Host的地位是完全对等的。从外部看，VM就像一台真实的机器一样，有自己的MAC和IP。这样极大地简化了网络结构。
 不过，在学校的网络环境下，每个IP需要登录web认证后才能访问外网，每个桥接的VM不能都需要登录，而每个账号只能同时登录一个IP的限制导致只能有一个台机器能连上外网，所以不适合采用桥接。

 另外，虽然名叫 **桥接**，但vbox和VMware都没有用到“网桥”。桥接实际是用所谓的网卡的“混合模式”实现的，即一个网卡可以伪装成不同MAC地址的多个网卡。在Hyper-V中倒是添加了一个虚拟网桥和一个虚拟网卡，功能上是一样的。
 KVM的各种网络连接方式都需要设置一个bridge，区别在于这个bridge与物理网卡（如eth0）的连接及路由设置。KVM的网络连接不是像vbox，VMware或Hyper-V那样由VMM实现的，而是利用了 **已有的** Linux的bridge-utils，TUN/TAP，iptables等功能。

<a name="docker-bridge">

# Docker的桥接模式

 >注意：docker的网络连接方式也有 **桥接**，但实际上它的工作模式是 **NAT网络**。docker在添加了一个docker0网桥，IP网段在`172.17.0.0`，如果外网要访问容器，需要做端口映射。这样的网络连接方式下，**不同Host** 的容器是不能直接通信的，这是个很大的局限。
 实际上docker也可以实现KVM那样的真正的 **桥接** ，具体可参考文章 [桥接模式构建 docker 网络](http://my.oschina.net/astute/blog/293944) 和[Four ways to connect a docker container to a local network](http://blog.oddbit.com/2014/08/11/four-ways-to-connect-a-docker/) 。看起来这样的连接方式还是比较实现容易的，但目前docker还没有在官方的实现中直接支持这种连接方式。

# 内部网络/LAN区段
 这种方式下只有连接到同一内部网络的VM之间能够通信，而VM与Host，与外网都不能通信。
 一个内部网络是由网络名来区分的。
 VMware和Hyper-V的内部网络可以设置网络带宽，丢包率等，方便进行网络方面的实验，不过作为其它场景的实验环境就不太合适了。
 内部网络不支持DHCP，可以专门添加一个多网卡的VM作为网络服务器，完成DHCP，网关，路由等功能。

# 选择哪种连接方式？
 + **NAT网络**			：优点是可以充分共享Host的网络连接，包括VPN；不足是Host和外部访问VM需要端口映射；VM的IP地址不受外部影响，适合搭建 **实验环境** 。
 + **Host-Only仅主机**	：优点是Host可以方便地访问VM；不足是不能访问外网；VM的IP地址也不受外部影响，如果NAT网络需要较多的端口映射，可以考虑每台VM设置2个网卡，一个工作在NAT网络模式，另一个工作在Host-Only模式。
 + **桥接**				：优点是设置简便，很容易实现互联互通；不足是不能共享Host的网络连接（web登录认证，VPN），而且VM的IP地址受外部影响，更换了网络环境，IP地址可能会发生变化。假设用笔记本电脑搭建的实验环境，VM使用桥接模式，在宿舍的IP地址和在会议室的IP地址是不同的，依赖IP的设置都要修改，是很不方便的。

 在 **数据中心** 里，网络环境不经常变化，**桥接** 模式屏蔽了VMM的影响，可以直接应用已有的交换机设备和配置，简化了网络配置操作。如果交换机支持VLAN，那么可以启用VLAN作为VM的网络隔离，虽然有不能超过4096（ $ 2^{12} $，12 bit）个VLAN的数量限制，但对企业内部的私有云场景应该是足够的。
 此外，虚拟网络还要考虑IP地址的容量，分配和管理方式避免冲突，多租户的隔离，以及故障转移后的IP重用策略等，了解的不多，这里就不多说了。


# Bonus

**[The Datacenter as a Computer - An Introduction to the Design of Warehouse-Scale Machines - 2e - 2013](/doc/The_Datacenter_as_a_Computer_An_Introduction_to_the_Design_of_Warehouse_Scale_Machines_2e_2013.pdf)**

![AWS 数据中心与 VPC 揭秘 - QCon Beijing 2017 - infoq](http://www.infoq.com/cn/presentations/aws-data-center-and-vpc-secret)

![Google的数据中心](/img/google_dc1.jpg)

![Google的数据中心](/img/google_dc2.jpg)