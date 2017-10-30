title: 局域网内的远程操作
date: 2016-09-18
category: [misc]
tags:

---
一些基础的远程操作，包括ssh，共享文件，远程桌面。

<!--more-->

---

<!-- TOC -->

- [Linux远程执行命令（ssh）](#linux%E8%BF%9C%E7%A8%8B%E6%89%A7%E8%A1%8C%E5%91%BD%E4%BB%A4%EF%BC%88ssh%EF%BC%89)
    - [Linux下设置和使用ssh](#linux%E4%B8%8B%E8%AE%BE%E7%BD%AE%E5%92%8C%E4%BD%BF%E7%94%A8ssh)
    - [Windows安装和设置xshell，使用密码ssh登录](#windows%E5%AE%89%E8%A3%85%E5%92%8C%E8%AE%BE%E7%BD%AExshell%EF%BC%8C%E4%BD%BF%E7%94%A8%E5%AF%86%E7%A0%81ssh%E7%99%BB%E5%BD%95)
    - [scp](#scp)
    - [sftp](#sftp)
    - [设置ssh使用密钥登录](#%E8%AE%BE%E7%BD%AEssh%E4%BD%BF%E7%94%A8%E5%AF%86%E9%92%A5%E7%99%BB%E5%BD%95)
        - [生成密钥对](#%E7%94%9F%E6%88%90%E5%AF%86%E9%92%A5%E5%AF%B9)
        - [分发密钥对](#%E5%88%86%E5%8F%91%E5%AF%86%E9%92%A5%E5%AF%B9)
            - [ssh-copy-id](#ssh-copy-id)
            - [复制密钥文本](#%E5%A4%8D%E5%88%B6%E5%AF%86%E9%92%A5%E6%96%87%E6%9C%AC)
        - [ssh config设置](#ssh-config%E8%AE%BE%E7%BD%AE)
        - [踢出ssh会话](#%E8%B8%A2%E5%87%BAssh%E4%BC%9A%E8%AF%9D)
- [远程共享文件（SMB/CIFS）](#%E8%BF%9C%E7%A8%8B%E5%85%B1%E4%BA%AB%E6%96%87%E4%BB%B6%EF%BC%88smbcifs%EF%BC%89)
    - [Samba访问Windows提供的共享文件](#samba%E8%AE%BF%E9%97%AEwindows%E6%8F%90%E4%BE%9B%E7%9A%84%E5%85%B1%E4%BA%AB%E6%96%87%E4%BB%B6)
    - [Windows访问Samba的共享文件](#windows%E8%AE%BF%E9%97%AEsamba%E7%9A%84%E5%85%B1%E4%BA%AB%E6%96%87%E4%BB%B6)
    - [更改Samba的默认端口号](#%E6%9B%B4%E6%94%B9samba%E7%9A%84%E9%BB%98%E8%AE%A4%E7%AB%AF%E5%8F%A3%E5%8F%B7)
- [远程桌面](#%E8%BF%9C%E7%A8%8B%E6%A1%8C%E9%9D%A2)
    - [mstsc](#mstsc)
    - [vnc](#vnc)
    - [其它](#%E5%85%B6%E5%AE%83)

<!-- /TOC -->

这里简单介绍局域网中Windows与Linux系统之间的一些基本远程操作，包括远程执行命令（`ssh`），共享文件（`Samba`）和远程桌面（`mstsc`和`vnc`）。
远程操作一般是“服务器-客户端”模式，有的服务程序或客户端是操作系统内置的，开箱即用，有的程序则需要手动安装。

为了方便配置，**建议关闭系统的防火墙**。下面例子使用的远程Linux主机是Ubuntu 16.04，IP是`10.1.1.5`，用户名`ying`；Windows的IP是`10.1.1.1`，用户名也是`ying`。

# Linux远程执行命令（ssh）
ssh（[Secure Shell](https://en.wikipedia.org/wiki/Secure_Shell)）通过加密的网络通道在客户端和服务器之间传递命令及其输出。
在ssh之前，远程执行命令是通过`telnet`或`rsh`等程序实现的，数据是明文传输的，缺乏安全性。ssh提供了一个在网络上认证用户和加密数据的通道，执行命令是直接使用的系统内置的shell。
可以在ssh提供的加密通道上完成其它网络通信：如`scp`是在ssh加密通道上实现的远程拷贝（`rcp`）；`sftp`是在ssh加密通道上实现的`ftp`；`git`也有使用ssh加密通道传输文件的模式。
对于Linux这种主要通过shell命令行交互的系统来说，使用ssh远程登录到服务器上，就跟直接在机器上敲命令就没什么区别了。

## Linux下设置和使用ssh
Linux系统的ssh服务程序是`OpenSSH Server`，执行命令`sudo apt install openssh-server`。
安装过程中会将ssh服务程序（`sshd`）添加为开机启动的系统服务，默认设置允许当前用户通过密码登录ssh。

> 查看SSH Server状态，执行`systemctl status sshd`
> 启动，停止或重启服务，执行`sudo systemctl start/stop/restart sshd`

一般Linux系统都内置了ssh客户端，执行
`ssh 用户名@主机名或IP`
登录到远程主机（如果用户名与当前登录的用户名相同，可以省略）。
登录到本机的命令是`ssh localhost`
第一次登录某个主机会提示是否 **信任** 该主机，需要输入`yes`，之后才会提示输入远程主机的登录密码。

>修改`/etc/ssh/ssh_config`，将其中`#   StrictHostKeyChecking ask` 改为 `StrictHostKeyChecking no`，这样在第一次登录时就不会询问是否要信任该主机了。

如果登录到远程主机只是执行一两条命令，可执行
`ssh 用户名@主机名或IP 命令`
当然，每次还是需要输入密码，参考下面的设置密钥登录后就方便多了。

## Windows安装和设置xshell，使用密码ssh登录
Windows目前没有内置的ssh客户端，可以安装Putty、SecureCRT、xshell等ssh客户端软件，或者使用Cygwin/MinGW，git（包含MinGW），Bash on Windows等附带的ssh命令。

> Windows版的`git`包含一个简化版`MinGW`，将`<git安装目录>\usr\bin`这个路径添加到Windows的`Path`环境变量，就可以在Windows的命令窗口执行`ssh`，`scp`及其它很多Linux命令了。
> `MinGW` 中也包含SSH Server程序`sshd`，不过估计很少会登录到Windows执行命令行操作吧。

从官网下载并安装 [xshell](http://www.netsarang.com/download/down_xsh.html) （需要注册一个免费的账号，选择免费的Home/School许可），也可以在百度搜索“xshell”，第一条结果即是，注意要选择 **普通下载**。

启动xshell后，可以直接执行`ssh ying@10.1.1.5`，会提示输入密码，首次连接也会提示“未知的主机密钥”，选择保存即可。
![](/img/xshell-ui.png)

> 工具栏的打开会话按钮，可以从其中选择某个会话，也可以直接在xshell中执行`open <会话名>`。
> 工具栏的那个带小齿轮的按钮是“默认会话属性”，修改其中的设置会影响新建的会话。
> 每个会话即`<用户文档>\NetSarang\Xshell\Sessions`下的一个配置文件，会话也可以复制后修改。

为方便后续使用，可以为这个虚拟机创建一个会话。单击工具栏的“新建”按钮，在打开的 **“会话属性”** 对话框中
+ 在“连接” 输入主机 `10.1.1.5`，在用户身份验证中选择方法为Password，输入用户名 `ying` 和 密码；
+ 在“终端” 修改“编码”为UTF-8；
+ 在“外观” 修改终端字体和配色方案，我比较习惯黑底绿字的配色，使用Consolas字体，使用闪烁的光标。

![](/img/xshell-prop.png)

> 注意：Windows的快捷键与Linux终端的快捷键存在冲突，如“复制”`Ctrl+C`对应的是中断当前命令。
> xshell中默认“复制”、“粘贴”的快捷键是`Ctrl+Ins`，`Shift+Ins`，而不是`Ctrl+C`，`Ctrl+V`。
可以打开 “工具”->“选项”，“键盘和鼠标”选项卡，“按键对应”->“编辑”，将其修改为`Ctrl+C`，`Ctrl+V`，而原来Linux终端的快捷键需要加`Shift`，如中断当前命令的`Ctrl+C`变成了`Ctrl+Shift+C`。

## scp
通过ssh加密的通道传输文件。文件路径格式为`用户名@主机名或IP:主机上的路径`。注意，Windows文件路径中的盘符`C:\`变成了`/c/`。
{% codeblock line_number:false%}
scp ying@10.1.1.5:/home/ying/.ssh/id_rsa.pub /c/users/ying/.ssh/
{% endcodeblock %}

## sftp
`OpenSSH Server`内置了一个`sftp`服务器，会随`sshd`服务自动启动。我们还需要一个`sftp`的客户端即可传送文件。
这里使用图形界面的，跨平台的，免费的，开源的[Filezilla](https://filezilla-project.org/download.php?type=client)。下载并安装后，在“快速连接”工具栏输入主机`sftp://10.1.1.5`，及用户名 `ying` 和密码，端口为22，单击“快速连接”，然后就可以进行文件传输和管理了。
![](/img/sftp.png)

Android上的`ES文件浏览器`也支持`sftp`（还支持下面介绍的smb局域网文件共享）。

## 设置ssh使用密钥登录
更安全而且方便的ssh登录方式是使用密钥对(key)。密钥对包含公钥和私钥，其实是两个很长的整数（被编码为字符串）。比如采用`rsa`算法，公钥和私钥分别保存在两个 **文本** 文件`id_rsa.pub`和`id_rsa`中。
+ 公钥`id_rsa.pub`保存在要登录的目标机器上（服务器，Github等），
+ 私钥`id_rsa`保存在 **发起** 登录的机器上（客户端），私钥要妥善保管，防止泄露。

Linux主机的密钥对默认保存在`~/.ssh/`目录。
Windows是`C:\Users\<Win用户名>\.ssh\`目录。在图形界面的文件管理器中不能创建以`.`开头的文件夹，需要在命令窗口操作：打开Windows的命令窗口（`Win键+X，C`），执行命令`mkdir C:\Users\<Win用户名>\.ssh`。

### 生成密钥对
因为加密算法是公开的，有多种工具可以生成密钥。
对Linux或MinGW，执行`ssh-keygen -t rsa -P ""` ，会在`~/.ssh/`生成密钥对`id_rsa.pub`和`id_rsa`。

xshell也可以生成密钥对：
+ 打开 “工具”-> “新建用户密钥生成向导” 或 “工具”-> “用户密钥管理者” -> “生成” 生成一个密钥类型为RSA的密钥，向导的最后一步会显示公钥，可以将其保存起来；
+ 选择刚创建的密钥，单击“导出”按钮，保存私钥，默认的格式与OpenSSH相同；
+ 选择刚创建的密钥，单击“属性”按钮，在“公钥”选项卡中保存公钥。

![](/img/xshell-key.png)

### 分发密钥对
要启用密钥，需清除其它用户访问私钥的权限（600），并公钥拷贝到远程目标Linux主机的`.ssh/authorized_keys`文件中。
分发密钥对其实就是在在Windows和Linux之间传送文件，可以使用上面提到的`scp`和`sftp`，也可以参考后面要介绍的smb文件共享；或者更复杂一些，搭建一个Web服务器，把文件放到上面，在Linux执行`wget`或`curl`命令下载，Windows可以通过浏览器下载。下面还有另外两种方法。

#### ssh-copy-id
执行命令`ssh-copy-id -i 公钥文件 用户名@主机名或IP`，将公钥拷贝到远程Linux主机的`/home/<用户名>/.ssh/authorized_keys`文件中。当然，这个命令需要用密码访问远程主机。

#### 复制密钥文本
如将Linux主机上生成的私钥`id_rsa`拷贝到Windows上：
+ 使用xshell用密码登录到Linux，执行`cat ~/.ssh/id_rsa`，输出私钥的内容，复制输出的文字。
+ 在Windows文件管理器中打开路径`C:\Users\<Win用户名>\.ssh`，在其中创建一个名为`id_rsa.txt`的文本文件，将上一步复制的文字粘贴进去，然后把文件名的`.txt`扩展名去掉，即改为`id_rsa`。

可以参考上面的方式将Windows上生成的公钥拷贝到远程Linux主机上。因为还要从远程Linux主机上执行`git`、`ssh`等命令，所以也要把私钥放拷过去。当然，也可以使用不同的密钥对。

### ssh config设置
在`~/.ssh/config`文件中可以设置多个远程主机的别名，地址，端口，用户名和密钥，简化ssh命令。
{% codeblock line_number:false%}
Host   别名
    HostName 主机名或IP
    Port     端口
    User     用户名
    IdentityFile   私钥文件

Host u
    Hostname  10.1.1.5
    Port      22
    User      ying

Host          10.1.1.6
Port          2222

{% endcodeblock %}

这样就可以直接执行`ssh 别名`登录指定的主机，而且不同的主机可以使用不同的端口，用户，密钥配置。别名还可以用在`scp`的路径中。

### 踢出ssh会话
查看在线用户：`w` 或 `who`，两者输出格式有所不同。
查看自己的连接信息：`who am i`。
踢出其它会话：`pkill -9 -t pts/1 `，其中`pts/1`是被踢会话的终端。

# 远程共享文件（SMB/CIFS）
“共享文件”（[Server Message Block，SMB](https://en.wikipedia.org/wiki/Server_Message_Block) )，改进的版本称为Common Internet File System，CIFS），是Windows上为局域网用户提供的远程访问文件的功能。Windows内置了smb的服务程序和客户端。
Samba是Linux上实现SMB/CIFS协议的开源服务程序及客户端。
Linux上与SMB/CIFS功能是类似的是“网络文件系统”（[Network File System，NFS](https://en.wikipedia.org/wiki/Network_File_System) ）。SMB和NFS功能相似，都是文件级别（相比于块级别iSCSI等方式）的远程存储服务。Windows默认没有安装NFS功能，但可以通过`控制面板→程序和功能→启用或关闭Windows功能`来添加NFS客户端和服务端软件。

> 注意：只能共享某个文件夹，不能单独共享某个文件。Windows会限制能链接的共享用户数量，如果需要提供共享文件服务，Samba是更好的选择。
> 共享配合文件系统的权限设置，可以实现精细的权限控制，比如 “只能上传，不能下载，不能删除” 这样的需求（上传作业的文件服务器）。

Linux一般内置了smb的客户端（mount.cifs模块）。如果没有，可以执行`sudo apt install cifs-utils`来安装。

## Samba访问Windows提供的共享文件
Windows上启用共享文件夹只要在文件夹的`属性对话框→共享选项卡→高级共享`中设置即可，在这个对话框中还可以指定用户和读写权限。共享名和实际的文件夹名可以不同。如果在共享文件名后添加`$`，就表示是隐藏的，必须通过输入完整路径才能打开。
![Windows上启用共享文件夹](/img/win-share.png)

创建挂载点`mkdir ~/z`，并在`/etc/fstab`中添加
{% codeblock line_number:false%}
//10.1.1.1/文档 /home/ying/z cifs username=Win用户名,password=Win密码,uid=1000,rw,iocharset=utf8,sec=ntlm 0 0
{% endcodeblock %}

执行`sudo mount -a`，挂载`/etc/fstab`中新增的设置。
执行`ls ~/z`，应列出共享文件夹中的内容，确认挂载成功。

> 上面的命令将共享文件夹`文档`挂载到Ubuntu的`/home/ying/z`，有读写权限。因为设置了终端编码为UTF-8，中文的文件名也能正常显示。
> 其中uid是Ubuntu中用户`ying`的，具体的值可执行命令`id`，或在`/etc/passwd`中查看。

## Windows访问Samba的共享文件
先要安装`Samba File Server`，执行`sudo apt install samba samba-common`。
> 查看Samba Server的运行状态，执行`systemctl status smbd`
> 启动，停止或重启服务，执行`sudo systemctl start/stop/restart smbd`

添加共享文件夹：执行 `sudo nano /etc/samba/smb.conf`，在末尾添加如下内容，添加了只读的根目录`/`和可读写的`/home/ying`目录，但显示为`all`和`ying`。
{% codeblock line_number:false%}
[all]
    comment = fs root directory
    path = /
;   writeable = no
;   browseable = yes
    valid users = ying

[ying]
    comment = ying's home
    path = /home/ying
    writeable = yes
    create mask = 0664
    directory mask = 0775
;   browseable = yes
    valid users = ying
{% endcodeblock %}

将`ying`添加为smb的共享用户：`sudo smbpasswd -a ying`， 按提示设置`ying`的smb密码，**可以与系统密码不同**。
重启smbd，使设置生效：`sudo systemctl restart smbd`。

> Samba的权限问题：Samba中的用户需要是Ubuntu已有的用户，还要给Samba的用户设置相关文件和目录的读写权限。

从Windows的文件管理器的地址栏访问 `\\10.1.1.5` ，会看到刚添加的两个共享文件夹。可以在文件夹上右击，快捷菜单中有 **“映射网络驱动器”** 的选项，也可以像普通文件夹一样创建快捷方式。除了IP地址，还可以通过Ubuntu的机器名来访问，若机器名为u，则地址为`\\u` 。

从macOS和Ubuntu访问共享文件（不论Windows或Ubuntu提供的）的路径格式是`smb://10.1.1.5`或`smb://u`。macOS会自动把共享文件挂载到`/Volumes`下。

> Windows可以通过机器名来访问Ubuntu是因为Samba默认开启了局域网内的`WINS`名字服务。
> 另一种方法是在`hosts`文件中为IP地址指定名字。

![](/img/smb.png)

> Samba共享文件与`sftp`的区别在于，`sftp`不能直接编辑文件，必须要把文件拷贝下来后才能处理，而操作共享文件跟本机的文件没有太大区别。
> PS, `testparm`命令可以用来检查`smb.conf`的配置是否正确。

> 参考
+ [Setting up Samba as a Standalone Server - samba wiki](https://wiki.samba.org/index.php/Setting_up_Samba_as_a_Standalone_Server)
+ [在CentOS 7中Samba服务安装和配置](http://lybing.blog.51cto.com/3286625/1676515)

## 更改Samba的默认端口号
2017年5月份的勒索病毒WanaCrypt会扫描开放445文件共享端口的Windows设备，导致网络管理员禁封了445端口。如果客户端和服务器在同一局域网，通讯都是在二层，不会受到影响，可以正常使用共享文件，但如果经过路由器，就不能使用了。实际中发现即便是在同一个局域网，另外的实验室也无法访问我们实验室的共享文件，可能两个实验室各自的交换机又连到一个三层交换机上了吧。
估计445一封了之，是不会再有解封之日了。好在还可以变通一下，修改Samba的端口号，绕过封锁。不爽的是，Windows的默认端口号是无法修改的，而Linux，macOS，Android的ES文件管理器都支持指定端口号，地址格式是`smb://10.1.1.5:4455/home/`，其中4455是修改后的端口号。

修改Samba的端口号只需在`/etc/samba/smb.conf`中增加
{% codeblock line_number:false%}
[global]
   smb ports = 4455 445  # 可以同时监听多个端口号
...
{% endcodeblock %}

# 远程桌面
## mstsc
Windows除家庭版之外均内置了远程桌面服务和客户端，使用的是远程桌面协议[Remote Desktop Protocol，RDP](https://en.wikipedia.org/wiki/Remote_Desktop_Protocol)。

+ 客户端在`所有程序→Windows附件→远程桌面连接`，或直接执行命令`mstsc`。
+ 服务程序：依次打开`控制面板→所有控制面板项→系统`，或`Win+X，Y`，然后单击左侧的`高级系统设置`，打开`系统属性`对话框，在`远程`选项卡中的`远程桌面`部分选中`允许远程连接到此计算机`，并选择某个用户。
![Windows上的远程桌面客户端](/img/win-mstsc.png)
![Windows上启用远程桌面](/img/win-mstsc-svr.png)

Ubuntu桌面版内置了可以访问Windows远程桌面的客户端；安卓和iOS系统也有远程桌面的App，但这三个系统都没有远程桌面的服务程序，Windows无法通过mstsc远程连接到它们的图形界面。
如果是在安卓平板或iPad上使用远程桌面连接到Windows系统，那么Windows会自动切换到触屏模式，就相当于在使用一个Windows系统的平板了，当然是台式机的性能。

Windows远程桌面一般只支持单个用户访问，如果有用户在使用远程桌面，那么本地的就会锁屏；但是服务器版可以设置支持多个用户同时使用远程桌面，彼此都是独立的窗口。

## vnc
VNC（[Virtual Network Computing](https://en.wikipedia.org/wiki/Virtual_Network_Computing)）是Linux上的远程桌面共享协议。Linux下有多个桌面环境，如Gnome，KDE，unity，xfce等，VNC对不同桌面系统的支持不同。此外，VNC的客户端及服务端也有多种实现，如x11vnc、realvnc、tigervnc、tightvnc、ultravnc等。Ubuntu Unity下自带了`远程共享`程序实现了VNC功能。由于vnc远比ssh占用的网络带宽大，而Linux上的大部分操作可以通过ssh来执行，所以不推荐使用VNC。
VNC默认桌面会话使用5900端口，可以开启多个桌面会话，新的VNC桌面会话的端口号依次增加。与Windows的远程桌面不同，VNC在远程访问时不会锁屏，而是同步显示默认桌面会话的显示。

可以参考教程[https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-vnc-on-ubuntu-16-04] ，在Ubuntu上安装和配置xfce桌面及tightvnc服务端。

Windows上没有内置的VNC客户端，有一些免费的`VNC-Viewer`程序。

> 注意： 按上面的设置启用VNC后，使用的是xfce桌面环境，
> 默认的 `Tab` 键补全终端命令与窗口管理的快捷键冲突，需要在“Settings-> Window Manager -> Keyboard”中清除 `Switch Window from same application` 关联的快捷键
> 还可以在 “Settings-> Keyboard -> application shortcut” 中设置打开终端的快捷键 `exo-open --launch TerminalEmulator ~ Ctrl+Alt+T`

## 其它
在局域网之外，如果网络连接比较复杂，mstsc或vnc可能都无法穿过机构强制的防火墙。有一些远程访问软件，比如 **[TeamViewer](https://www.teamviewer.com)**，[向日葵](http://sunlogin.oray.com/zh_CN/)等，可以实现广域网情形的远程访问，前提是两端的机器都能访问Internet公网，而mstsc和vnc则不要求必须能够访问公网。另一种方法是申请机构内的VPN。
