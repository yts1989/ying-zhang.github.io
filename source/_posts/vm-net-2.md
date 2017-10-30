title: 再说docker及云中的网络连接
category: [cloud]
tags:
date: 2017-04-08
---
之前写过一篇[关于虚拟机和docker网络的日志](https://ying-zhang.github.io/cloud/2016/vm-net/)，主要介绍的是VM的虚拟网络，顺带提了一下docker的桥接。
经过2016年的几次版本升级，docker的网络功能有了很大的改善，已经基本稳定可用了。
去年[与一个大四做毕设的学弟折腾过一段时间docker网络](https://github.com/NAP-GHJ/NetTool/blob/master/blog/%E4%BD%BF%E7%94%A8%E6%89%8B%E5%86%8C.md)，这里参考浙大SEL实验室的《Docker容器与容器云 第2版》（下面简称《容器云》书） **4.2节 Docker高级网络实战** ，整理一下相关内容。

<!--more-->

---

<!-- TOC -->

    - [title: 再说docker及云中的网络连接](#title-%E5%86%8D%E8%AF%B4docker%E5%8F%8A%E4%BA%91%E4%B8%AD%E7%9A%84%E7%BD%91%E7%BB%9C%E8%BF%9E%E6%8E%A5)
- [再说Docker桥接模式，路由器，NAT，交换机](#%E5%86%8D%E8%AF%B4docker%E6%A1%A5%E6%8E%A5%E6%A8%A1%E5%BC%8F%EF%BC%8C%E8%B7%AF%E7%94%B1%E5%99%A8%EF%BC%8Cnat%EF%BC%8C%E4%BA%A4%E6%8D%A2%E6%9C%BA)
- [Docker跨主机的网络](#docker%E8%B7%A8%E4%B8%BB%E6%9C%BA%E7%9A%84%E7%BD%91%E7%BB%9C)
- [跨主机网络的实现机制](#%E8%B7%A8%E4%B8%BB%E6%9C%BA%E7%BD%91%E7%BB%9C%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%9C%BA%E5%88%B6)
- [macvlan](#macvlan)
- [overlay网络](#overlay%E7%BD%91%E7%BB%9C)
- [对容器网络的需求](#%E5%AF%B9%E5%AE%B9%E5%99%A8%E7%BD%91%E7%BB%9C%E7%9A%84%E9%9C%80%E6%B1%82)
- [其它](#%E5%85%B6%E5%AE%83)
- [脑洞](#%E8%84%91%E6%B4%9E)

<!-- /TOC -->

# 再说Docker桥接模式，路由器，NAT，交换机
[上篇日志提到Docker的桥接模式（bridge）实际上是NAT方式](https://ying-zhang.github.io/cloud/2016/vm-net/index.html#docker-bridge)，也给了两个设置Docker“名副其实”的bridge模式的链接，这点在《容器云》的 “4.2.2 pipework 原理解析：1. 将Docker配置到本地网络环境中” 一节也提到了。
实际为NAT的docker bridge模式和类似VM的bridge模式都是用到了Linux bridge虚拟网桥。这个虚拟设备其实既是一个 **虚拟路由器**，也是一个 **虚拟交换机**，因为（家用）路由器的一侧就包括了一个交换机。
参考下图。
![家用无线路由器拓扑](/img/wifi-router.png)
+ 如果把墙上的网口用网线接到路由器的 **WAN口**，把家里的PC，手机等接到路由器的 **有线LAN口** 或 **wifi**，这是路由器的正常工作模式，即 **NAT模式**，类似于 **docker的bridge模式**。
+ 如果稍稍开一下脑洞，把网线接到 **有线LAN口中的任意一个**，留着WAN口什么也不接，这时PC，手机等设备还是能连到网络上的！这时只用了路由器的LAN一侧，只工作在 **交换机** 模式，这类似于 **VM的bridge模式**。其实交换机的功能就是把一个网络端口变成多个网络端口。

# Docker跨主机的网络
上面强调docker的bridge模式名不副实，其实是因为docker早期版本缺少跨主机的网络功能，造成了诸多不便。在“传统的”VM技术中，是使用bridge模式来实现不同物理主机上VM直接联网的，即将VM网络接入到主机网络环境中。docker虽然用了同样的称呼，但没有提供同样的功能，除了端口映射和host模式，没有办法方便地将不同物理主机上的容器互相连通，让docker的用户非常痛苦。
多个第三方工具，比如pipework、weave、socketplane、flannel，calico等都是为了实现docker的跨主机网络。最终，在docker 1.9，docker提供了内置的Overlay网络模式。

回顾一下docker支持的网络模式 https://docs.docker.com/engine/userguide/networking/ ：
+ none：docker撒手不管，由用户或第三方工具提供网络功能，上节中pipework就是这样的工具；
+ host：直接使用主机的网络栈，即没有网络隔离；
+ bridge：NAT模式，容器可以使用主机的网络访问外部，但外部访问容器的应用需要做端口映射，即在`docker run`命令中提供`-p`参数
+ overlay, gwbridge：这是ver 1.9引入的覆盖网络，下面会单独介绍。

>docker还有一种`container`模式，使用已有容器的网络。Kubernetes的Pod是依赖于这种网络模式的：一个Pod包括多个功能相关的容器，它们共用一个网络栈，是 [**史上最小的docker镜像** `pause` ](https://github.com/kubernetes/kubernetes/tree/master/build/pause)创建的。
>新版本（我使用的17.05.0-ce版）的docker使用这种模式的命令参数是`docker run --net container:<network-container-name> <image> <entrypoint>`

# 跨主机网络的实现机制
跨主机网络需要处理：
+ 容器和主机网络的拓扑：即哪个容器子网对应于哪台物理主机。拓扑信息的源头当然是容器启动命令，可以通过标准路由协议在各主机的网络组件守护进程之间交换信息，也可以将拓扑信息写入etcd这样的高可用集中存储。
+ 数据包的转发：iptables，VXLAN等。
+ 子网隔离：对跨主机的网络，除了实现不同主机上的容器之间能够联网互通，还要能 **不通**，属于不同子网的容器之间不能通讯。

不同的实现机制，在功能和性能上有所区别：
+ 将容器置于主机网络中：类似上面提到的VM的bridge模式，及macvlan方式；这种方式需要 **外部机制** 来支持子网，一般是传统的 **VLAN**，即交换机端口设置不同的VLAN ID；VLAN的12位长度限制了子网的数量，对公有云平台是一个限制，但对企业内的私有云一般是够用了。
+ 路由转发：如`calico`，仍然使用docker的bridge模式，但不再使用NAT，而是 **路由转发**；NAT的出现是解决私有子网（`10.0.0.0/8， 172.16.0.0/12， 192.168.0.0/24`），所以路由器对这几个IP网段特别处理了。如果把这3个网段当成普通的IP网段，容器网络和主机网络就跟普通的多层IP网络是一样的（当然主机可以直接通讯，多数情况不必经过物理的核心路由器）。这样 **虚拟交换机** 就不够用了， 需要 **虚拟路由器**，`iptables` 就是这样一个虚拟路由器（软件路由器，也被作为防火墙）。可以手动添加iptables的路由条目；也可以使用工具自动化这个过程，比如通过标准的路由协议，或开发非标准的方式。
+ overlay网络：docker内置了overlay支持，CoreOS的flannel也是早期比较常用的overlay网络工具，Kubernetes就是使用的flannel。overlay网络是将容器的数据包封装起来，当成普通的数据，到目的主机后再拆开，转发给对应的目的容器。

>路由转发和overlay方式都有一个 **限制**：每个物理主机上的容器子网必须在 **不同** 的网段。因为这2种方式都会用到docker网桥（docker0），网桥会聪明地通过子网掩码识别哪些数据包是它负责的本地LAN，只有不是同一子网的数据包才会从这个网桥发出去。
不过这个限制可以被绕过去。比如从calico或flannel的设置看起来，不同主机上的容器子网是在同一个IP网段的（比如`10.0.0.0/16`），但实际上每个主机上的容器子网是更细的网段，比如主机A上是`10.0.1.0/24`，主机B上是`10.0.2.0/24`，甚至使用了`10.0.0.1/32`这样的“子网”。

>早期docker在每个主机上只能设置一个bridge，同一主机上的容器彼此无法隔离。目前版本的docker可以设置多个网络。

# macvlan
Linux bridge是一个软件实现的虚拟网桥，而macvlan则利用了网卡硬件的支持。
目前的docker内置了macvlan支持。参考[docker的macvlan文档](https://docs.docker.com/engine/userguide/networking/get-started-macvlan/) 和 [数人云的一篇文档 - docker跨主机macvlan网络配置](https://github.com/alfredhuang211/study-docker-doc/blob/master/docker%E8%B7%A8%E4%B8%BB%E6%9C%BAmacvlan%E7%BD%91%E7%BB%9C%E9%85%8D%E7%BD%AE.md)，按下面的例子可以设置macvlan：
```
# 主机IP为10.1.1.10/24，网卡名称ens33，首先开启网卡的混杂模式
ip link set ens33 promisc on

# 创建名为macvlan_net的docker网络
docker network create -d macvlan --subnet=10.1.1.0/24 --gateway=10.1.1.2 -o parent=ens33 macvlan_net

# 运行容器时指定或动态分配IP。注意，动态分配的IP可能与主机IP冲突
docker run -ti --net macvlan_net --ip=10.1.1.101  centos:net /bin/bash
docker run -ti --net macvlan_net centos:net /bin/bash
```

上面使用的`centos:net`镜像是在centos:7基础上安装了`iproute`和`net-tools`软件包，以在镜像内提供`ip`，`ifconfig`等网络管理命令。
```
# Dockerfile
FROM centos:7
RUN yum install net-tools iproute -y

# docker build . -t centos:net
```

按上面的命令创建的2个容器彼此可以pin通，也可以ping通网关和其它主机，但 **不能ping同本主机**，这是macvlan本身的限制（摊手）。macvlan有4种工作模式，可以参考[linux 网络虚拟化： macvlan](https://cizixs.github.io/2017/02/14/network-virtualization-macvlan)、[Linux 上虚拟网络与真实网络的映射](https://www.ibm.com/developerworks/cn/linux/1312_xiawc_linuxvirtnet/)和[Some notes on macvlan/macvtap](http://backreference.org/2014/03/20/some-notes-on-macvlanmacvtap/)，分别为VEPA（不常用，需交换机支持hairpin模式，又称reflective relay反射中继），桥接，私有和Passthru（直通）。
使用macvlan网络的容器被置于主机网络中。子网隔离需要VLAN，但实际上同一主机上的容器都使用主机的网线，对应的是交换机的同一个端口，也就是同一个VLAN ID，所以需要macvlan作为一个虚拟交换机，也支持设置VLAN ID。这方面可以参考 [基于macvlan的Docker容器网络系统的设计与实现 - 浙江大学硕士学位论文 - 万方](http://t.cn/RXpdDrf) 和 [Virtual switching technologies and Linux bridge - ppt](/doc/Virtual_switching_technologies_and_Linux_bridge_ppt.pdf)。

# overlay网络
flannel开始是自己实现的overlay机制，将IP包封装到UDP的数据段，即IP in IP，需要拦截并应答容器的ARP包；后来加入了VXLAN的转发方式。VXLAN是将完整的二层以太网包封装到UDP的数据段，即MAC in IP，提供了完整的虚拟二层网络，而且VXLAN是内核支持的，性能好得多。
早期版本flannel的配置可以参考[一篇文章带你了解Flannel](http://dockone.io/article/618)。
关于[VXLAN](https://tools.ietf.org/html/rfc7348)，从名字上看似乎是VLAN的扩展，但它跟VLAN的实现方式有很大的差别。
VLAN是二层以太网包的一个段。对VLAN的支持和VLAN ID是在交换机上设置的（针对交换机各端口，计算机上不需要特别设置）。虽然把VLAN从12位拓展到24位似乎比较简单直接，但需要 **升级交换机** 才能支持新的以太网数据包格式。所以VXLAN把扩展的24位VLAN ID放到了UDP的数据段。VXLAN数据包的处理是通过虚拟的VTEP网卡实现的，实际上是发生在主机上，对交换机而言是透明的。交换机要支持VXLAN还比较复杂些，看起来VXLAN是为主机上的虚拟交换机量身定制的。VXLAN的下层网络可以手动设置点对点的拓扑，也支持通过组播自动发现并组网，不过这样就比较复杂了。
![](/img/vnet-vxlan.png)。

VXLAN设计的一个应用场景是在不同的三层网络（如多个数据中心）之上（即overlay）建立一个虚拟的二层网络，即所谓的“大二层”。大二层的需求是为了应对虚拟机的在线迁移。虚拟机一般使用桥接模式，与主机有相同的网络环境。当VM迁移到另一个主机后，VM的MAC和IP应保持不变，以便减少对VM内应用和其用户的影响（虽然VM的MAC和IP没有变化，但新主机对应于交换机的端口发生了变化，用户会经历一小段时间的网络中断，以等待交换机端口学习新的ARP映射）。如果不同主机在不同的VLAN甚至跨数据中心，那么迁移后VM将无法与原VLAN的节点通讯。所以在线迁移需要没有隔离的大二层网络，但这样会造成广播风暴。点对点的VXLAN overlay网络可以比较好的解决这个矛盾。
docker出现后，面临的跨主机网络与大二层需求不同但也有类似之处，docker内置的overlay网络就使用了VXLAN转发。
>VLAN作为传统的子网隔离机制，是工作在二层的。除了VLAN ID不同，每个VLAN的IP网段也不同，即三层通过IP子网来隔离。VLAN之间如果需要通讯，则需设置经路由转发。

VXLAN是由VMware为主提出来的，微软提出了类似的[NVGRE](https://tools.ietf.org/html/rfc7637)。NVGRE的数据包格式与VXLAN相比向下兼容性不够好，需要升级设备才能应用。

>参考
+ [【华为悦读汇】技术发烧友：认识VXLAN](http://support.huawei.com/huaweiconnect/enterprise/thread-334207.html)
+ [【华为悦读汇】技术发烧友：闲话大二层网络](http://support.huawei.com/huaweiconnect/enterprise/thread-333013.html)
+ [Overlay 网络技术，最想解决什么问题？](https://www.zhihu.com/question/24393680)
+ [基于容器云平台的网络资源管理与配置系统 - 浙江大学硕士论文 - 万方](http://t.cn/RXpdege)

# 对容器网络的需求

+ 提供类似传统网络的体验
  + VPS（Virtual Private Server）× n + 虚拟网络 = VPC（Virtual Private Cloud）：不同租户的子网彼此隔离，租户可以指定或被分配IP网段，DHCP或静态指定IP，关联公网IP以便与互联网连通；
  + 租户可以有多个子网，设置虚拟路由器；
  + 安全组，防火墙，负载均衡，DNS；
+ 性能：高带宽，低延迟，扩展性。VXLAN和calico是目前性能比较好的2种技术。
+ 容器与物理主机，虚拟机互联共存：这一点目前还没有比较好的实现。

# 其它
关于Calico，可参考[其官网](http://docs.projectcalico.org/v2.1/getting-started/docker/)和[将Docker网络方案进行到底](http://blog.dataman-inc.com/shurenyun-docker-133/)。
关于不同网络方式的性能，可参考豆瓣上（是的，豆瓣）的这篇[Docker network on cloud 中文](https://www.douban.com/note/530365327/)或者 https://cmgs.me/life/docker-network-cloud 。这里盗个图。
![](/img/vnet-pk.png)

# 脑洞
上面的一堆方案，有的自称为SDN，但实际与SDN还有些差距。SDN是针对网络设备的，而这些方案多是在服务器主机上实现虚拟网络设备（交换机，路由器），更接近NFV的目标。
**被上面一堆方案搞得头痛的，不妨看看这样一个有意思的想法：[Jumpers and the Software Defined Localhost](https://coreos.com/blog/jumpers-and-the-software-defined-localhost.html)：容器中只有一个loop（127.0.0.1）网卡！完全由外部来管理容器网络。**
隔离是肯定没问题，都不需要子网的概念了，但都只有一个`127.0.0.1`的IP，如何与其它容器通讯呢？jumpers使用的是端口，域名应该更好些，Docker 已经为每个容器内置了一个[DNS（127.0.0.11）](https://www.infoq.com/news/2016/08/docker-service-load-balancing)来帮助实现服务发现。


[DockOne微信分享（一三零）：探究PaaS网络模型设计](http://dockone.io/article/2504)