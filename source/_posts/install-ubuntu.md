title: 安装Ubuntu Server 16.04 lts
date: 2016-09-16
category: [misc]
tags:

---
记录一下安装Ubuntu Server 16.04 lts 及 基本设置作为备忘。

<!--more-->

---

<!-- TOC -->

- [说明](#%E8%AF%B4%E6%98%8E)
    - [为什么是16.04](#%E4%B8%BA%E4%BB%80%E4%B9%88%E6%98%AF1604)
    - [为什么是Server版](#%E4%B8%BA%E4%BB%80%E4%B9%88%E6%98%AFserver%E7%89%88)
    - [为什么使用 VM](#%E4%B8%BA%E4%BB%80%E4%B9%88%E4%BD%BF%E7%94%A8-vm)
- [在虚拟中安装Ubuntu](#%E5%9C%A8%E8%99%9A%E6%8B%9F%E4%B8%AD%E5%AE%89%E8%A3%85ubuntu)
    - [Virtualbox的全局设定](#virtualbox%E7%9A%84%E5%85%A8%E5%B1%80%E8%AE%BE%E5%AE%9A)
    - [安装Ubuntu](#%E5%AE%89%E8%A3%85ubuntu)
- [基本配置](#%E5%9F%BA%E6%9C%AC%E9%85%8D%E7%BD%AE)
    - [设置网络](#%E8%AE%BE%E7%BD%AE%E7%BD%91%E7%BB%9C)
    - [系统更新和升级](#%E7%B3%BB%E7%BB%9F%E6%9B%B4%E6%96%B0%E5%92%8C%E5%8D%87%E7%BA%A7)
    - [[可选] 切换为root用户](#%E5%8F%AF%E9%80%89-%E5%88%87%E6%8D%A2%E4%B8%BAroot%E7%94%A8%E6%88%B7)
    - [[可选] 设置sudo免密码](#%E5%8F%AF%E9%80%89-%E8%AE%BE%E7%BD%AEsudo%E5%85%8D%E5%AF%86%E7%A0%81)
    - [杂项设置](#%E6%9D%82%E9%A1%B9%E8%AE%BE%E7%BD%AE)
    - [Tips](#tips)
        - [PATH环境变量](#path%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F)
        - [`which`命令](#which%E5%91%BD%E4%BB%A4)
        - [查看系统版本](#%E6%9F%A5%E7%9C%8B%E7%B3%BB%E7%BB%9F%E7%89%88%E6%9C%AC)

<!-- /TOC -->

# 说明

## 为什么是16.04

+ 16.04是lts版，相比非lts要稳定一些(不会瞎折腾)；虽然14.04也是lts版，但毕竟2014年都已经过去很久了。
+ 16.04的软件源更新比较快，而且可以用更简洁的`apt`命令代替`apt-get`来安装程序。
+ 另一个原因是[systemd](https://zh.wikipedia.org/wiki/Systemd)。`systemd`参考了Mac OS X的`launchd`，是一个替代`init`程序的系统服务管理组件。其它发行版，如RHEL/CentOS，CoreOS都已经采用了`systemd`。Ubuntu 14.04 lts采用的是`upstart`，直到15.04版才转用`systemd`。在运行一些 **dokcer**，**etcd** 等服务程序的示例时，经常会看到用`systemd`的unit文件定义的系统服务(service)，不能直接拿来用在Ubuntu 14.04 lts。当然在15.04版及以后的版本就可以了，由于一些默认路径不同，不同发行版的unit文件可能还是需要适当修改。

## 为什么是Server版
因为在Ubuntu下只是在 **敲命令**，极少有非使用图形界面不可的情况，又何必忍受Ubuntu臃肿的GUI呢。ssh远程登录到系统上，所有操作都通过命令行搞定，不必费心某个软件没有Linux版，或者QQ、输入法的功能弱，也不用被 **Ubuntu桌面版默认强制** 安装一堆充满bug的办公软件，多媒体软件等。

## 为什么使用 VM
如果有两台机器，一个装Ubuntu Server，连上网络然后扔在一边（只是一个机箱，都不需要显示器、键盘鼠标），另一个装Windows，远程访问即可。
如果只有一台机器，那么用Virtulbox创建虚拟机，安装一个Server版，平时可以选择“无界面启动”(`headless`)，没有多余的窗口，然后就跟使用物理服务器一样ssh远程登录到系统上。

> 如果通过ssh远程执行命令，就不必使用vbox的虚拟机窗口了。启动虚拟机时，可以选择“无界面启动”，也可以在Windows命令行启动虚拟机
"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" startvm <虚拟机名> --type headless
其中`C:\Program Files\Oracle\VirtualBox\`是vbox的安装路径，可以把它添加到`PATH`环境变量中。
![](/img/vbox-headless.png)

使用VM比物理机器更好的地方，
+ 一是可以使用主机的网络，对学校网络这种外网帐号只能在一处使用的场景很方便；
+ 另一点是可以利用VM的 **快照功能** 方便地进行全系统的备份和恢复。

VM相比物理机器的性能损失，或者较少的CPU、内存资源其实影响并不大，毕竟云计算中都在普遍使用VM嘛。
即便是Mac OS X这样的Unix环境，也建议使用VM安装Ubuntu，除了上面两个优点，还可以防止误操作弄乱系统，另外可以避免Mac OS X内置命令与Linux不兼容的困扰。
不建议在物理机器上安装双系统。

# 在虚拟中安装Ubuntu

## Virtualbox的全局设定
安装好Virtualbox后，可以在`管理->全局设定`中修改`默认虚拟电脑位置`；另外在`网络`中添加`Nat网络`，并修改IP网段；修改已经默认添加的`仅主机(Host-Only)网络`的IP网段。
如下图，设置Nat网络的IP网段为`10.0.1.0/24`，设置Host-Only网络的`主机虚拟网络界面`(即vbox的虚拟网卡)的IP为`10.1.1.1`，并启用DHCP服务器，设置其IP网段。当然使用Host-Only虚拟网卡默认的 `192.168.56.0`网段也是可以的。这里是为了说明如何设置任意的（私网）IP网段，二是为了以后敲命令时IP较简短。
![添加Nat网络，并修改IP网段](/img/vboxconf-NatNetwork.png)

![修改默认Host-Only网络的IP网段](/img/vboxconf-hostonly.png)

## 安装Ubuntu
从Ubuntu官网下载[Server 16.04.2 的.iso镜像](http://releases.ubuntu.com/16.04/ubuntu-16.04.2-server-amd64.iso)。
在vbox中新建Linux类型虚拟机，选择Ubuntu 64位，虚拟磁盘使用vhd格式，动态扩展，将下载的.iso镜像挂载到虚拟机，启动虚拟机后开始安装。
![](/img/vboxsetting-store.png)

+ **[可选]** 安装系统前先 **不接入网络**，即在VM设置中不勾选`启用网络连接`，以免安装过程中联网更新耗时较长。
+ **[强烈建议]** 安装时语言选择为 **英文**：如果选择默认语言为中文，安装后一些命令会显示中文的帮助信息，结果因为终端不能显示中文而变成乱码，造成不便。英文系统也是使用UTF8编码，可以在ssh客户端正常显示中文。但`Location`要选择实际的区域，以匹配正确的时区。

在最后的步骤中选中`Samba File Server`，`Standard System Utility`和`OpenSSH Server`（`空格键`选择或取消选择，`回车键`确认并继续下一步）。

>如果安装前没有接入网络，则安装后关闭虚拟机，添加“NAT网络”和“仅主机(Host-Only)网卡”两个网卡。

![](/img/vboxsetting-net.png)

Server没有图形界面，启动系统，进入的是下面这样一个黑乎乎的界面。输入安装时设置的用户名（这里是`ying`），回车，输入密码（没有任何显示），回车，进入系统的`shell`。
![](/img/vm-tty.png)

# 基本配置

## 设置网络
如果安装时没有接入网络，添加网卡后需要在`/etc/network/interfaces`中设置一下。
Ubuntu 16.04的网卡命名不是以前的类似`eth0`，`eth1`，而是`enp0s?`这样。`?`代表一个数字，
执行`ip a`，如上图，输出的`2: enp0s3...`和`3: enp0s8...`就是已经识别出的网卡名。

设置网卡：执行`sudo nano /etc/network/interfaces`，使用`nano`编辑`/etc/network/interfaces`，**增加** 下面的内容
{% codeblock line_number:false%}
auto  enp0s3       # NAT网络，用于连接外网
iface enp0s3 inet dhcp

auto  enp0s8       # Host-Only网卡，设置静态IP，内网
iface enp0s8 inet static
address 10.1.1.5
netmask 255.255.255.0
{% endcodeblock %}

按 `Ctrl+X` 快捷键，再按 `Y` 键，回车，保存并退出`nano`。

>`nano` 是一个简单的命令行文本编辑器，功能比较弱，但比`vim`直观一些。

启动网卡：执行`sudo ifup enp0s3 enp0s8`，再次执行`ip a`，可以看到这两个网卡已经获取了IP地址。

确认网卡工作正常：
+ Nat网络连接外网：执行`curl ip.cn`，应返回一个IP地址和乱码（乱码是IP地址对应的中文的地理位置）
+ Host-only连接内网（主机）：执行`ping 10.1.1.1 -c 5`，应该能ping通。

> 网络工作正常后，就可以通过ssh登录到虚拟机来执行命令了。

## 系统更新和升级
{% codeblock line_number:false%}
sudo apt update   # 更新apt
sudo apt upgrade  # 升级系统，可能耗时较长。
{% endcodeblock %}

如果网络速度较慢，可将apt源更换为[国内163](http://mirrors.163.com/.help/ubuntu.html)的，或者[http://mirrors.nju.edu.cn]，其中Ubuntu 16.04的代号是`xenial`。


## [可选] 切换为root用户
安装系统时设置的的用户有`sudo`权限。直接使用`root` **不是** 一种好的做法，不过可以省去很多权限相关的问题和很多命令前面的`sudo` 。
执行`sudo passwd`，先输入当前用户的密码以授权`sudo`，然后输入两次`root`的登录密码。执行`exit`注销当前用户，以`root`和刚才设置的密码登录。

ssh默认禁止`root`用密码登录。可以修改`/etc/ssh/sshd_config`允许`root`使用密码登录，或为`root`设置使用密钥登录。

## [可选] 设置sudo免密码
如果不想直接使用`root`用户，还可以设置执行`sudo`时 **免输密码**。执行`sudo visudo`(实际上是`nano`编辑器)，找到 `%sudo	ALL=(ALL:ALL) ALL`这一行，改为`%sudo	ALL=(ALL:ALL) NOPASSWD: ALL` 。

## 杂项设置

{% codeblock line_number:false%}
; 安装常用软件
sudo apt install git zsh tree

; 设置Git的全局用户名和Email
git config --global user.name  "ZHANG Ying"
git config --global user.email "your@email.com"

; 设置Git换行符转换规则，input选项会把Windows下的CRLF换行符转换成Linux下的LF
; 参考https://git-scm.com/book/be/v2/Customizing-Git-Git-Configuration
git config --global core.autocrlf input  # true会将LF转换为CRLF，false则不做任何处理

; 设置zsh和oh-my-zsh
sudo chsh ying -s /usr/bin/zsh
wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | sh
; 使用短网址  
wget https://git.io/SM81Wg -O - | sh

; 注销后重新登录，默认的shell已经从bash（dash）切换到了zsh。

; 以下修改~/.zshrc，执行 source ~/.zshrc 应用更改
; 禁用oh-my-zsh的自动更新，取消下面一行的注释
DISABLE_AUTO_UPDATE="true"

; 默认已禁用oh-my-zsh的PATH，可在/etc/enviroment改PATH

; 增加alias
alias cls="clear"
alias dir="ls -alF"
alias ipconfig="ifconfig"
alias ping="ping -c 3"

alias ll="ls -alF"
alias ps="ps -af"
alias netstat="netstat -nap"
alias json="python -m json.tool"
{% endcodeblock %}

## Tips
### PATH环境变量
Linux下的绝大多数命令，其实是对应着一个可执行程序的 **文件名**。这些可执行文件分散在 `PATH` 环境变量设置的目录列表中，比如
{% codeblock line_number:false%}
echo $PATH
; 输出为
; /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
{% endcodeblock %}

可以查看这些目录中都有哪些可执行程序，也就是系统支持的命令了。比如Ubuntu系统安装后已经内置了Python2，Python3，Ruby，Perl，它们的可执行程序都在`/usr/bin`，可以通过`ls /usr/bin`来确认。
如果在上面的目录中都找不到要执行的程序文件名，那么就会得到 *无法找到命令* 的错误。如果不是敲错了命令，那么就需要完整的程序路径，或者将程序所在目录加到`PATH`。
当前目录（`.`）默认没有加入到`PATH`中去，这是出于安全考虑。要执行当前目录下的程序或脚本，需要`./foo.sh`这样。

### `which`命令
{% codeblock line_number:false%}
$ which which
which: shell built-in command

$ which echo
echo: shell built-in command

$ which python
/usr/bin/python
{% endcodeblock %}

### 查看系统版本
查看发行版本和代号，执行`cat /etc/os-release`
查看内核版本，执行`uname -a`

-----

设置完成后，关闭VM，在vbox中创建一个快照，另外可以再将虚拟磁盘压缩备份。
![](/img/vbox-snapshot.png)
