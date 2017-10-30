title: 使用Cobbler搭建PXE服务器
category: misc
date: 2017-3-22
tags:

---
[PXE](https://zh.wikipedia.org/wiki/%E9%A2%84%E5%90%AF%E5%8A%A8%E6%89%A7%E8%A1%8C%E7%8E%AF%E5%A2%83) （Preboot eXecution Environment，预执行环境）是通过 **局域网** 来启动计算机（和安装操作系统）的技术。
一般是通过刻录到光驱或U盘的Live CD这样的本地存储来安装系统的，要通过网络来安装系统，首先要知道安装文件存放的服务器（TFTP服务器，Trivial File Transfer Protocol，精简FTP），而系统启动时网卡的IP都还没有。所以PXE必须要有一个 **DHCP（Dynamic Host Configuration Protocol，动态主机设置协议）**，不但负责为机器分配 IP地址，还会告知安装文件所在的服务器的 IP地址。
Cobbler简化了安装配置DHCP、TFTP、关联Kickstart应答文件等搭建PXE服务器的过程。

<!--more-->

---

<!-- TOC -->

- [测试环境](#%E6%B5%8B%E8%AF%95%E7%8E%AF%E5%A2%83)
- [Cobbler设置步骤](#cobbler%E8%AE%BE%E7%BD%AE%E6%AD%A5%E9%AA%A4)
    - [禁用SELinux](#%E7%A6%81%E7%94%A8selinux)
    - [禁用防火墙](#%E7%A6%81%E7%94%A8%E9%98%B2%E7%81%AB%E5%A2%99)
    - [安装软件](#%E5%AE%89%E8%A3%85%E8%BD%AF%E4%BB%B6)
    - [设置为开机启动的服务](#%E8%AE%BE%E7%BD%AE%E4%B8%BA%E5%BC%80%E6%9C%BA%E5%90%AF%E5%8A%A8%E7%9A%84%E6%9C%8D%E5%8A%A1)
    - [生成加密的root密码](#%E7%94%9F%E6%88%90%E5%8A%A0%E5%AF%86%E7%9A%84root%E5%AF%86%E7%A0%81)
- [修改cobbler的配置](#%E4%BF%AE%E6%94%B9cobbler%E7%9A%84%E9%85%8D%E7%BD%AE)
    - [修改xinetd tftp的配置](#%E4%BF%AE%E6%94%B9xinetd-tftp%E7%9A%84%E9%85%8D%E7%BD%AE)
    - [修改dhcp配置](#%E4%BF%AE%E6%94%B9dhcp%E9%85%8D%E7%BD%AE)
    - [使用 dnsmasq 提供 dhcp 服务](#%E4%BD%BF%E7%94%A8-dnsmasq-%E6%8F%90%E4%BE%9B-dhcp-%E6%9C%8D%E5%8A%A1)
    - [启动相关服务](#%E5%90%AF%E5%8A%A8%E7%9B%B8%E5%85%B3%E6%9C%8D%E5%8A%A1)
    - [执行 `cobbler check`](#%E6%89%A7%E8%A1%8C-cobbler-check)
    - [导入CentOS系统的iso安装镜像](#%E5%AF%BC%E5%85%A5centos%E7%B3%BB%E7%BB%9F%E7%9A%84iso%E5%AE%89%E8%A3%85%E9%95%9C%E5%83%8F)
    - [修改Kickstarts文件](#%E4%BF%AE%E6%94%B9kickstarts%E6%96%87%E4%BB%B6)
    - [更新设置，最后的检查](#%E6%9B%B4%E6%96%B0%E8%AE%BE%E7%BD%AE%EF%BC%8C%E6%9C%80%E5%90%8E%E7%9A%84%E6%A3%80%E6%9F%A5)
- [在其它机器使用PXE安装系统](#%E5%9C%A8%E5%85%B6%E5%AE%83%E6%9C%BA%E5%99%A8%E4%BD%BF%E7%94%A8pxe%E5%AE%89%E8%A3%85%E7%B3%BB%E7%BB%9F)
- [在docker容器中运行cobbler](#%E5%9C%A8docker%E5%AE%B9%E5%99%A8%E4%B8%AD%E8%BF%90%E8%A1%8Ccobbler)
    - [docker容器使用systemd](#docker%E5%AE%B9%E5%99%A8%E4%BD%BF%E7%94%A8systemd)
    - [使用容器的几个坑](#%E4%BD%BF%E7%94%A8%E5%AE%B9%E5%99%A8%E7%9A%84%E5%87%A0%E4%B8%AA%E5%9D%91)
- [真机上部署遇到的问题](#%E7%9C%9F%E6%9C%BA%E4%B8%8A%E9%83%A8%E7%BD%B2%E9%81%87%E5%88%B0%E7%9A%84%E9%97%AE%E9%A2%98)
    - [选择合适的网卡](#%E9%80%89%E6%8B%A9%E5%90%88%E9%80%82%E7%9A%84%E7%BD%91%E5%8D%A1)
    - [Dell iDRAC](#dell-idrac)
- [参考](#%E5%8F%82%E8%80%83)

<!-- /TOC -->

# 测试环境
虚拟机使用NAT网络，IP段10.1.1.0，子网掩码：255.255.255.0，DNS/DHCP/Gateway：10.1.1.2。

新建一个虚拟机，安装CentOS 7，使用的镜像是 **[CentOS-7-x86_64-Minimal-1611.iso](http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1611.iso)** 。因为是Minimal的，不必选择附加的软件包。
安装后登录系统，查看IP地址`ip a`，可以通过ssh登录到VM。
关闭VM，拍摄一个快照。
+ IP ： 10.1.1.10
+ 子网掩码：255.255.255.0
+ 用户：root

# Cobbler设置步骤

> 因为虚拟机已经加载了iso镜像，可以在Guest OS中直接挂载。对远程的服务器，可以配置的同时把iso镜像拷贝到远程机器上去，或者使用共享文件，节省一点拷贝文件的时间。

## 禁用SELinux
```
sed -i 's/SELINUX\=enforcing/SELINUX\=disabled/g' /etc/selinux/config
setenforce 0
```

## 禁用防火墙
```
systemctl disable firewalld
systemctl stop firewalld
```

## 安装软件
```
yum -y install epel-release
yum -y install cobbler dhcp httpd xinetd pykickstart fence-agents
```

## 设置为开机启动的服务
```
systemctl enable cobblerd dhcpd httpd rsyncd tftp xinetd
```

> PXE 需要从 dhcp服务器获取新的IP地址，以及TFTP，HTTP服务器的IP地址， TFTP服务器提供操作系统的安装文件，HTTP服务器提供Kickstart应答文件。
cobbler_web 是cobbler的设置界面，跟HTTP服务不是一回事，可不必安装。

## 生成加密的root密码
```
openssl passwd -1 -salt "centos" "centos"
# 输出，由于salt和密码都是指定的，这段加密的字符串也就是确定的了
$1$centos$Uq6E6Wp5SDZYbs6MCmamP0
```

# 修改cobbler的配置
```
vi /etc/cobbler/settings
# 改动 的内容如下
default_password_crypted: "$1$centos$Uq6E6Wp5SDZYbs6MCmamP0"
manage_dhcp: 1
next_server: 10.1.1.10
server: 10.1.1.10
```
> [修改后的完整 settings 文件 (删除了注释)](/doc/cobbler-setting.txt)

## 修改xinetd tftp的配置
```
vi /etc/xinetd.d/tftp
# 将 disable = yes 改为  disable = no
```

## 修改dhcp配置
```
vi /etc/cobbler/dhcp.template
# 改动的内容包括 子网段，分配的IP区间，
# 以及 默认网关（路由器），DNS（默认网关和DNS的设置不影响PXE装机过程，只是新装的机器启动后可能无法访问外网）
# $next-server 是指向 /etc/cobbler/settings 中的对应值
#
subnet 10.1.1.0 netmask 255.255.255.0 {
     option routers             10.1.1.2;
     option domain-name-servers 10.1.1.2;
     option subnet-mask         255.255.255.0;
     range dynamic-bootp        10.1.1.100 10.1.1.110;
     default-lease-time         21700;
     max-lease-time             43100;
     next-server                $next_server;

     class "pxeclients" {
          match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
          if option pxe-system-type = 00:02 {
              filename "ia64/elilo.efi";
          } else if option pxe-system-type = 00:06 {
              filename "grub/grub-x86.efi";
          } else if option pxe-system-type = 00:07 {
              filename "grub/grub-x86_64.efi";
          } else {
              filename "pxelinux.0";
          }
     }
}
```

> 如果遇到DHCP服务启动失败，可能是 dhcpd 先于 cobblerd 启动，导致 cobbler 来不及设置 dhcpd ，dhcpd.conf 配置文件还是空的。
遇到这种情况，可以将 `/etc/cobbler/settings` 中的 `manage_dhcp: 1` 改为 `0`， 然后手动管理 dhcp：修改 `/etc/dhcp/dhcpd.conf`，格式如下。

```
subnet 10.1.1.0 netmask 255.255.255.0 {
        option routers 10.1.1.2;
        option domain-name-servers 10.1.1.2;
        option subnet-mask 255.255.255.0;
        default-lease-time 21600;
        max-lease-time 43200;
        range 10.1.1.100 10.1.1.110;
        next-server 10.1.1.10;
        filename "pxelinux.0";
}
```
注意要把 `$next_server` 设置为 **具体的 IP地址**，这个配置项是 PXE 的关键。
<!--这样的一个好处是可以设置 **多个** `subnet {...}` 段，以针对不同的网络环境，dhcpd会自动将 subnet段 与系统网卡的配置匹配，忽略不匹配的设置。
很可惜 cobbler 的 next_server 只能指定一个 IP， 换了机器还要修改设置。-->
如果dhcp服务器（也就是cobbler的服务器）有多个网卡，上面dhcp的配置项与哪个网卡的IP段匹配，就在这个网卡所在的局域网上提供 dhcp 服务。如果没有找到任何匹配的网卡， dhcpd 会报错退出。
如果所在的局域网已经有其它的dhcp服务器，那么会存在竞争，可以考虑使用 dnsmasq。
`/etc/cobbler/settings` 中 `next_server` 和 `server` 指定的IP地址要和DHCP的网段在 **同一个局域网段** 才能正常工作。

## 使用 dnsmasq 提供 dhcp 服务

参考[Managing DHCP - cobbler manual](http://cobbler.github.io/manuals/2.8.0/3/4/1_-_Managing_DHCP.html)，如果 dhcpd 无法正确配置，可以使用 dnsmasq。首先需要安装： `yum install -y dnsmasq`。
如果让cobbler来配置dnsmasq，需要设置`/etc/cobbler/settings` 中为 `manage_dhcp: 1`，
然后修改 `/etc/cobbler/modules.conf`，将
```
[dhcp]
module = manage_isc   # isc 即 dhcpd
# 改为
[dhcp]
module = manage_dnsmasq
```

然后修改 `/etc/cobbler/dnsmasq.template`，dnsmasq的配置比较简单，只要修改IP区间即可
```
dhcp-range=10.1.1.100,10.1.1.110
```

> 注意 dhcpd 与 dnsmasq 的区间格式不同，配置文件的格式错误会导致服务无法启动。
如果只是临时提供 dhcp 服务，可设置比较小的区间，**特别要避免与已有的重要服务器发生 IP地址 冲突**！

## 启动相关服务
> 注意： 重启服务器后，需要确认下面的几项服务是否正常启动。

```
systemctl start cobblerd dhcpd httpd rsyncd tftp xinetd
```

## 执行 `cobbler check`
```
# 输出如下，可见只有一项问题，因为安装的是CentOS系统，可以忽略这一项。
The following are potential configuration items that you may want to fix:
1 : debmirror package is not installed, it will be required to manage debian deployments and repositories
Restart cobblerd and then run 'cobbler sync' to apply changes.
```

+ 如果提示需下载额外的boot loader，可执行 `cobbler get-loaders` ，这 **不是必须的**，因为已经自带了常用系统的loader。如果确实要下载，且要使用代理的话，`/etc/cobbler/settings` 中可以在 `proxy_url_ext` 设置代理地址。
+ 如果输出中说 httpd无法访问 或 SELinux 没有关闭，执行`systemctl status httpd` 查看，还可以通过访问 http://10.1.1.10 （next_server的IP地址）来确认.可能是 `/etc/cobbler/settings` 中 `next_server` 或 `server` 的 IP地址 设置错误，改正IP地址后尝试重启 httpd。

再次执行 `cobbler check`，并检查相关的服务是否正常启动，然后继续执行下面的步骤。

## 导入CentOS系统的iso安装镜像

虚拟机中已经为Guest OS加载了iso文件到`/dev/cdrom`，需要再挂载到`/mnt`：`mount -t auto -o loop,ro /dev/cdrom /mnt`；
也可以直接挂载iso文件：`mount -t iso9660 -o loop,ro /your/path/to/CentOS-7-x86_64-Minimal-1611.iso /mnt`。

导入安装镜像：`cobbler import --name=centos --arch=x86_64 --path=/mnt`。
等几分钟才能执行完import，因为这一步把安装光盘拷贝到了 `/var/www/cobbler/ks_mirror/`

查看导入的项目：`cobbler profile list`
输出为 centos-x86_64，进一步查看，`cobbler profile report --name=centos-x86_64`。

## 修改Kickstarts文件
从上面的命令输出可知使用的Kickstart文件在 `/var/lib/cobbler/kickstarts/sample_end.ks`
修改下面2项
+ `firewall --enable`            改为 `firewall --disable`
+ `timezone  America/New_York`   改为 `timezone  Asia/Shanghai`

## 更新设置，最后的检查
```
cobbler sync
systemctl restart cobblerd
cobbler check
```

# 在其它机器使用PXE安装系统
新建一个虚拟机，不要直接启动，而是通过菜单选择“虚拟机 -> 电源 -> 打开电源时进入固件”，在虚拟机的BIOS中将网络启动设置为第一项，然后按 `F10` 键保存并重启虚拟机。
![](/img/pxe-boot-config.png)

稍等一下，DHCP配置完成后会显示启动项如下：
![](/img/pxe-boot-menu.png)
选择第二项 centos-x86_64，回车，就会开始自动安装。

稍等一会儿，安装完成后自动重启。可以按上面的步骤，在BIOS中修改默认启动项为本机硬盘。
从本机硬盘启动后，以root用户登录，密码就是之前使用openssl设置的密码，查看一下分配的IP地址，可以用ssh登录后继续系统管理操作。

> 从Cobbler官网的手册来看，它支持profile和system命令，应该是支持多网卡和多种操作系统的Profile等复杂需求的；
此外还可以在[settings](/doc/cobbler-setting.txt)中设置加入LDAP domain，代理，安装软件包，配置用户ssh key等等功能，以及构建定制的系统iso镜像，
在Kickstart文件中也可以完成硬盘分区，安装软件包，配置用户等等功能。
水平有限，就不深入这些功能了;-(

-----
# 在docker容器中运行cobbler

## docker容器使用systemd
通过 `docker pull centos:7` 直接pull下来的镜像 **不能使用** `systemd`， 因为这与 docker 的 **单容器单进程** 哲学不相容~~
我们需要自己build一个支持systemd的镜像，参考 [Docker Hub 的 CentOS镜像](https://hub.docker.com/_/centos/) 或[Docker Store 的  CentOS](https://store.docker.com/images/d5052416-4069-4619-8597-ba61df35ba6f)的页面中 **Systemd integration** 一节提供的 Dockerfile 即可。

再此镜像之上再build一个cobbler的镜像，这里偷懒，cobbler的Dockerfile只是安装安装必要的软件（**增加 which 和 curl**），启用相关服务项，其它设置进入容器的shell手工修改。

假设这个镜像的 tag 为 cobbler:default，启动一个容器，使用
+ host 网络，`--network host`
+ 特权模式，`--privileged`
+ 挂载cgroup的fs，以使用systemd，`-v /sys/fs/cgroup:/sys/fs/cgroup:ro`
+ 挂载`/mnt`，这是已经挂载到主机的CentOS安装文件iso镜像，`-v /mnt:/mnt:ro`

```
docker run -d --privileged --network host -v /sys/fs/cgroup:/sys/fs/cgroup:ro -v /mnt:/mnt:ro \ 
--name cobbler cobbler:latest /usr/sbin/init
```

因为容器的`Entrypoint`是`/usr/sbin/init`，而且是`-d`，即detached，启动后不会进入shell。
可以通过`docker exec`执行容器的shell，需要增加 `-ti`选项为shell分配一个终端，
```
docker exec -ti cobbler bash
```

退出容器（不会停止容器运行）的快捷键是 Ctrl + P, Q，或在容器的shell中执行`exit`。

按上述步骤启动容器，在容器的shell中执行上一节的操作，修改配置，导入iso镜像，检查各项服务是否正常启动。

## 使用容器的几个坑
> Docker官方的CentOS镜像太精简了，
+ 没有 `curl` 及 `wget`，还少了 `which`， 也需要装上；
+ `/etc/httpd/logs`是一个链接，但应该是一个目录，结果导致 httpd 无法启动。把原来的链接删掉，新建一个`/etc/httpd/logs/`目录即可；
+ 没有 `/var/log/cobbler/tasks` 目录，导致cobbler sync 失败，手动创建该目录。 


# 真机上部署遇到的问题
## 选择合适的网卡
服务器有2个网卡，BIOS默认只有一个网卡能通过PXE启动机器。一般的机器都是`em1`，但有的机器需要修改BIOS选择`em2`网卡才行。

## Dell iDRAC
服务器是Dell的，有2种型号，分别通过iDRAC6 和 iDRAC8 web界面远程管理。它们都可以提供服务器的画面显示，鼠标和键盘控制，并且可以将本地的iso镜像/光驱挂载为远程服务器的虚拟光驱。
在使用PXE之前，先要装好一台机器。
装机之前，先把iDRAC的固件升级了一下，通过机器的服务标签可以搜索到对应的固件。
iDRAC6 的画面显示是通过 jnlp控件 显示的，要安装好 jre，并在控制面板的Java选项（或直接执行`javacpl.exe`）中添加 `安全例外项`。jnlp控件都是一次性的，每个会话都要重新下载，还要点击n个安全提示对话框。最好使用IE，使用Chrome出现过安装系统快结束时页面错误，功亏一篑。
> iDRAC6 的 jviewer jnlp控件对应的jar包证书比较旧了，如果使用较新的jre，会因为证书过期而无法执行jnlp控件，所以需要安装 java 7 版本的 jre。

通过jnlp安装CentOS不管什么版本都是图形化的安装界面，太耗资源了，先要黑屏等着等传过去一堆文件之后才能显示出安装界面来，在安装界面虽然鼠标指针可以移动，但单击没有反应，键盘也没有反应;-( 
安装Ubuntu Server版基于的文本安装界面，很快就可以显示出来，可以只用键盘操作。

旧版iDRAC8 的jnlp同样是有显示但无法操作，好在升级后可以使用 html5 的新界面，而且支持多个会话。


# 参考
+ [Cobbler Quick Start](http://cobbler.github.io/manuals/quickstart/)
+ [Centos7.2安装Cobbler 并安装系统](http://readshlinux.blog.51cto.com/9322509/1812402)
+ [How to Install and Configure Cobbler on CentOS 7.x](http://www.linuxtechi.com/install-and-configure-cobbler-on-centos-7/)
