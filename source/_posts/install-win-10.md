title: 安装和设置Windows 10
date: 2016-09-17
category: [misc]
tags:

---
记录一下安装Windows 10 LTSB 及 基本设置作为备忘。

<!--more-->

<!-- TOC -->

- [为什么是 LTSB](#%E4%B8%BA%E4%BB%80%E4%B9%88%E6%98%AF-ltsb)
- [创建USB安装盘，安装系统](#%E5%88%9B%E5%BB%BAusb%E5%AE%89%E8%A3%85%E7%9B%98%EF%BC%8C%E5%AE%89%E8%A3%85%E7%B3%BB%E7%BB%9F)
- [硬盘分区规划](#%E7%A1%AC%E7%9B%98%E5%88%86%E5%8C%BA%E8%A7%84%E5%88%92)
- [常用快捷键](#%E5%B8%B8%E7%94%A8%E5%BF%AB%E6%8D%B7%E9%94%AE)
- [系统设置](#%E7%B3%BB%E7%BB%9F%E8%AE%BE%E7%BD%AE)
- [安装Linux常用工具](#%E5%AE%89%E8%A3%85linux%E5%B8%B8%E7%94%A8%E5%B7%A5%E5%85%B7)
- [Windows Server 2016 的设置](#windows-server-2016-%E7%9A%84%E8%AE%BE%E7%BD%AE)
- [Reg文件](#reg%E6%96%87%E4%BB%B6)
- [其它](#%E5%85%B6%E5%AE%83)
    - [开启虚拟wifi](#%E5%BC%80%E5%90%AF%E8%99%9A%E6%8B%9Fwifi)

<!-- /TOC -->

# 为什么是 LTSB
1、Win10 LTSB企业版没有Edge浏览器
2、无应用商店
3、无任何系统自带磁贴程序
4、无Cortana
5、在系统更新方面，用户能完全手动控制更新，选择和决定自己要的更新和驱动，更新内容和更新时间可以随意控制，但不能无限期推迟。
总结起来就是微软官方精简版啊。而且学校的KMS可以正常激活2015版的LTSB。
此外，还可以使用有学校后缀的Email在微软的[https://imagine.microsoft.com/zh-cn] 认证一个学生帐号，即可免费获得一个Windows Server 2016版的密钥。Win Server也是精简的，还可以彻底卸载Defender。

# 创建USB安装盘，安装系统
搜索`cn_windows_10_enterprise_2016_ltsb_x64_dvd_9060409.iso` 或在[http://msdn.itellyou.cn] 找到下载的ED2K链接，下载后用[Rufus](http://rufus.akeo.ie/) 制作USB启动安装盘，注意要 **取消** `高级选项` 中的 `使用Rufus MBR配合 BIOS ID...`。
然后重启机器，选择从U盘启动，按界面一步步安装系统即可。新建的用户一定要设置密码，方便后面远程操作。

# 硬盘分区规划
买一个SSD，只分一个区，即C盘，系统装在上面，以后的应用程序一般也按默认路径装在C盘。机器自带的机械硬盘，也只分一个区，保存个人数据。如果只有一块硬盘，也只要分一个系统分区和一个数据分区即可。多分无用，平添麻烦。如果需要Linux，装在虚拟机里，不要搞双系统。

# 常用快捷键
+ Win + D : 切换显示桌面
+ Win + E : 打开`文件资源管理器`
+ Win + R : 打开`运行`对话框
+ Win + X : 打开`开始`按钮的右键快捷菜单，*基本* 对应于`C:\Users\Ying\AppData\Local\Microsoft\Windows\WinX`目录的快捷方式
  + F : 程序和功能（即弹出上面的菜单后，松开手，再按小写的 `F` 键，或写成`Win+X, F`，这些键都在每个菜单项后面标出来了）
  + P : 控制面板
  + Y : 系统属性
  + C/A : 命令窗口
  + G : 计算机管理
  + U, I : 注销
  + U, R : 重启
  + U, U : 关机

> [向Win+X添加快捷方式](http://blog.sina.com.cn/s/blog_a0c06a350102y239.html)。为了方便使用git附带的minGW，创建了一个指向`git-bash.exe`的快捷方式。可以在`C:\Users\Ying\.bash_profile`增加一个`cd /E/Code`的命令来修改`git-bash`的默认启动目录。

# 系统设置

0. 安装系统更新（会自动安装大部分驱动）
1. 在`系统属性`对话框中
  + 更改计算机名
  + 调整视觉效果，禁用虚拟内存，修改默认启动项和等待时间，修改环境变量
  + 禁用系统保护
  + 禁用远程协助，开启远程桌面
2. 关闭休眠：Win+X, A，然后再命令行输入 `powercfg /h off`
3. 在每个分区，打开`分区属性`对话框，取消`除了文件属性外，还允许索引此驱动器上的文件内容`；磁盘清理，删除`Windows.old`
4. 安装字体，如`Microsoft Yahei Mono`，更改命令行窗口的默认字体（使用Microsoft Yahei Mono，见下面的注册表项）、颜色和半透明效果；更改记事本的字体
5. 更改IE主页为`about:tabs`，删除内置的加速器
6. 在`文件夹选项`
 + 设置`打开文件资源管理器时打开`此电脑
 + 取消隐私相关选项
 + `查看`中将大图标文件夹视图`应用到文件夹`
 + 取消`使用共享向导`
 + 取消`始终显示图标，从不显示缩略图`
 + 取消`隐藏受保护的操作系统文件`，选择`显示隐藏的文件、文件夹或驱动器`
 + 取消`隐藏已知文件类型的扩展名`
7. 在`个性化->主题->桌面图标设置`中选中显示`计算机`桌面图标
8. 任务栏上不显示搜索和任务视图按钮，使用小图标，显示所有托盘图标
9. 禁用防火墙，取消`安全性与维护`中的所有消息
10. 禁用Defender，：组策略，计算机配置，管理模板，Windows组件，Windows Defender
11. 禁用contana：组策略，计算机配置，管理模板，Windows组件，搜索
12. 安装`百度输入法`，设置使用双拼；安装`7zip`，`迅雷精简版`，`Everything`
13. 移动系统默认文件夹
  + 在D盘新建`D:\桌面`，`D:\下载`，`D:\App`，`D:\文档\音乐`，`D:文档\图片`，`D:\文档\视频`，`D:文档\收藏夹`
  + 在`C:\Users\Ying\`各系统默认文件夹的属性对话框的`位置`选项卡中，将其移动到对应的D盘新位置
  + `D:\App`存放一些绿色免安装的程序
  + **不要** 使用以前的更改注册表的方法来移动默认文件夹
14. 删除/合并开始菜单项
  + `C:\ProgramData\Microsoft\Windows\Start Menu`
  + `C:\Users\Ying\AppData\Roaming\Microsoft\Windows\Start Menu`
15. 禁用某些服务
16. 安装360，检查/清理系统，优化设置，然后卸掉；下载魔方PC Master绿色版，删掉自动更新等无关程序，使用`cleanmaster.exe`清理磁盘和隐私项，使用`winmaster.exe`清理右键菜单

Ghost制作系统备份。之后安装Office，Acrobat，QQ影音，Paint.net，xshell，Chrome，QQ，微信，百度云，Visual Studio Code，VMWare，MikTex，TexStudio，Jetbrains全家桶，打印机驱动等。

# 安装Linux常用工具
安装git（不是Github，当然也可以），其中自带`MinGW`，其中有`bash`、`curl`、`vim`、`ssh`等工具。
在`PATH`环境变量中添上其路径（`D:\App\git\mingw64\bin`，`D:\App\Git\usr\bin`和`D:\App\Git\bin`）即可。
还可以安装 [Linux Subsystem on Windows（WSL）](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide) 和 ‎Bash on Ubuntu on Windows。

# Windows Server 2016 的设置

+ 安装系统更新，鲁大师安装不能识别的设备驱动
+ 设置Windows audio服务为自动启动
+ 卸载Defender
+ 组策略-> 计算机配置
  + Windows设置-> 安全设置-> 帐户策略-> 密码策略-> 密码必须符合复杂性要求
  + Windows设置-> 安全设置-> 本地策略-> 安全选项-> 交互式登录：无须按Ctrl+Alt+Del
  + 管理模板-> 系统-> 显示“关闭事件跟踪程序”

# Reg文件

[直接下载Win10.reg](/doc/win10.reg)

```
Windows Registry Editor Version 5.00

;更改命令行窗口字体
[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Console\TrueTypeFont]
"0"="Lucida Console"
"936"="*Microsoft YaHei Mono"
"00"="Consolas"

;开启的分区共享(C$, D$...)
[HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System]
"LocalAccountTokenFilterPolicy"=dword:00000001

;清除通知区域图标，需重启文件资源管理器
[-HKEY_CURRENT_USER\Software\Classes\LocalSettings\Software\Microsoft\Windows\CurrentVersion\TrayNotify\PastIconsStream]
[-HKEY_CURRENT_USER\Software\Classes\LocalSettings\Software\Microsoft\Windows\CurrentVersion\TrayNotify\IconStreams]

;删除“此电脑”下的6个文件夹
;视频
[-HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\MyComputer\NameSpace\{f86fa3ab-70d2-4fc7-9c99-fcbf05467f3a}]
;文档
[-HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\MyComputer\NameSpace\{d3162b92-9365-467a-956b-92703aca08af}]
;桌面
[-HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\MyComputer\NameSpace\{B4BFCC3A-DB2C-424C-B029-7FE99A87C641}]
;音乐
[-HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\MyComputer\NameSpace\{3dfdf296-dbec-4fb4-81d1-6a3438bcf4de}]
;下载
[-HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\MyComputer\NameSpace\{088e3905-0323-4b02-9826-5d99428e115f}]
;图片
[-HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\MyComputer\NameSpace\{24ad3ad4-a569-4530-98e1-ab02f9417aa8}]

;删除文件资源管理器左侧的OneDrive
[HKEY_CLASSES_ROOT\CLSID\{018D5C66-4533-4307-9B53-224DE2ED1FE6}\ShellFolder]
"FolderValueFlags"=dword:00000028
"Attributes"=dword:f090004d

;使用“照片查看器”打开图片
[HKEY_CURRENT_USER\Software\Classes\.jpg]
@="PhotoViewer.FileAssoc.Tiff"
[HKEY_CURRENT_USER\Software\Classes\.jpeg]
@="PhotoViewer.FileAssoc.Tiff"
[HKEY_CURRENT_USER\Software\Classes\.gif]
@="PhotoViewer.FileAssoc.Tiff"
[HKEY_CURRENT_USER\Software\Classes\.png]
@="PhotoViewer.FileAssoc.Tiff"
[HKEY_CURRENT_USER\Software\Classes\.bmp]
@="PhotoViewer.FileAssoc.Tiff"
[HKEY_CURRENT_USER\Software\Classes\.tiff]
@="PhotoViewer.FileAssoc.Tiff"

```

# 其它
## 开启虚拟wifi
设置
```
netsh wlan set hostednetwork mode=allow ssid=Ying key=12345678
```

开启
```
netsh wlan start hostednetwork
```

连接外网
有线网卡 的属性中选择`共享`
