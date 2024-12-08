---
title: DD系统-服务器安全保护设置
mermaid: true
math: false
comments: true
hide: false
date: 2024-12-08 11:20:39
tags:
	- vps
categories:
	- VPS
---




# DD系统

基本所有的VPS商家，都会提供免费的Linux系统供安装，如CentOS、Debian、Ubuntu等。那为什么还要使用一键DD脚本重装/更换系统呢？

原因大概有这么几点：

- 商家提供的系统版本有限，可能没有自己需要的版本（如HostHatch只有CentOS只有7，Debian只有9、10）；
- 商家提供的系统大多都是改装过的，不纯净（如良心云、套路云服务器自带云镜、云盾等监控），可能存在软件兼容行问题；
- 商家提供的系统大多带有监控，虽说可以卸载，但是心里总是有疙瘩（指良心云、套路云）；针对以上几种情况，一键DD脚本就可以为服务器更换一个纯净的系统，帮你解决问题。

## 重要提醒

- 请仔细阅读服务器商家的ToS条款，事先确认你的服务器提供商是否支持你DD系统（重装自己的系统）由于授权问题，很多服务器提供商是禁止你把服务器DD成Windows系统的（比如Contabo），发现会暂停服务甚至删鸡！
- OpenVZ / LXC 架构系统不适用此脚本
- 注意重装有风险，请妥善备份好自己的数据，（阿里腾讯搬瓦工等有快照的商家，你可以先在后台存一个快照）可能导致无法开机（部分商家可以用VNC救回来，但本文不涉及），谨慎操作！
- 我买的是阿里云的香港轻量服务器，按照本文操作下来没有任何问题。

## 准备工作

使用脚本前最好先安装如下软件：(按需安装)

```shell
# CentOS 与 RedHat
yum install -y xz openssl gawk file wget

# Debian 与 Ubuntu
apt-get install -y xz-utils openssl gawk file
```

## 安装脚本

```shell
wget --no-check-certificate -O AutoReinstall.sh https://git.io/AutoReinstall.sh && bash AutoReinstall.sh
```

按照提示一路向下安装，不出意外的话，等待系统关机，关机后等到5-15分钟，重新连接，输入密码，即可得到一个纯净的Linux系统了。

💡注意：安装过程中会显示新系统的密码，（Centos默认密码`Pwd@CentOS`，其它系统`Pwd@Linux`）。

我安装的是Debian 11，所以接下来的安装以Debian 11为演示。

# Debian 11安全设置

## 毛胚房装修

ssh登陆系统（用户名`root`，密码`Pwd@Linux`）,输入`vim` ,`sudo` , `ifconfig`…，发现是要啥啥没有。是真“纯净”啊。也可能是我之前用Ubuntu用久了，第一次使用Debian。总之是，新系统我们需要按照自己的使用习惯安装自己常用的命令。接下来，我按照我的使用场景安装我所需的操作命令（由于是root权限下登陆的，就没加sudo，新系统下也没有，建议新系统用root权限设置好一切，关闭root远程登陆）：

```shell
# 更新系统包&升级现有软件
apt update 
apt upgrade 

# 安装防火墙命令
apt install ufw

# 安装sudo命令
apt install sudo

#安装vim
apt install vim

#安装ifconfig
apt install net-tools

#安装wget
apt install wget

#安装git
apt install git
```

大概就这些，自己需要的再添加。

## 更改SSH端口

1. 将默认的`22`的默认端口禁用掉，该用其他端口，我这里使用`222`端口:

```shell
vim /etc/ssh/sshd_config
```

命令行`/Port 22`找到22端口，如果这一行开头有个#，证明这一行【不生效】（被注释掉了），直接 把#删掉就好。

![](https://img.stuarthub.us.kg/2024/04/6f8d04edec5c673dbff3e3d2e8318b9f.png)

1. 然后`esc` `wq` ，退出。
2. 最后要做的事，是重启 ssh 服务，使变更生效

```shell
sudo service sshd restart
```

1. 最后的最后，要在云服务提供商后台的安全组/防火墙开放`66`端口。旧的22端口，喜欢留着就留着吧，当僚机。

注意：为了保证你不会失联，请不要关闭当前的ssh登录窗口！而是另外开一个窗口来测试！

注意：为了保证你不会失联，请不要关闭当前的ssh登录窗口！而是另外开一个窗口来测试！

注意：为了保证你不会失联，请不要关闭当前的ssh登录窗口！而是另外开一个窗口来测试！

![](https://img.stuarthub.us.kg/2024/04/22bcc57709b470fa5d2faf983b772c80.png)

## 禁止ping

```shell
sudo vim /etc/ufw/before.rules
```

搜索：`echo-request`，把`ACCEPT`改成`DROP`，然后保存

![](https://img.stuarthub.us.kg/2024/04/636b29fe2c420c7fa85423114d2294a1.png)

## 禁止暴力破解

安装Fail2ban

```shell
sudo apt update && sudo apt install fail2ban
```

```shell
cd /etc/fail2ban # 进入fail2ban目录
sudo cp fail2ban.conf fail2ban.local  # 复制一份配置文件
```

编辑我们的自定义配置 `vim fail2ban.local` ,末尾插入我们的ssh配置

```
默认的是没有配置的，我们加入一个配置

[sshd]
enable = ture
port = 66   # 改成自己对应的ssh端口
filter =sshd
logpath = /var/log/auth.log
maxretry = 5  #最大尝试5次
bantime = -1  # 打入黑名单
```

![](https://img.stuarthub.us.kg/2024/04/fe1e427bc1171da9bab3ce3f41196548.png)

```shell
sudo service fail2ban restart #重启
sudo fail2ban-client status #查看状态
sudo fail2ban-client status sshd #查看sshd的详细状态
```

如需解禁指定IP，命令如下：

```shell
sudo fail2ban-client set sshd unbanip 192.0.0.1 #解禁指定IP
```

## 开启防火墙

Ubuntu默认自己已经是自带ufw防火墙了，只是没有启动而已（如果是Debian的话，需要安装）

```shell
设置ufw使用默认值
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 66/tcp comment 'SSH'  # 允许我门自动移的ssh
sudo ufw allow http # 允许http
sudo ufw allow https # 允许https
sudo ufw enable  # 启动ufw防火墙
sudo ufw status #查看防火墙状态

sudo ufw status numbered # 显示防火墙规则序列
sudo ufw delete 5 #删除一条规则

sudo ufw reload #重载配置
```

## 使用复杂的密码登陆

### 插曲

在正式密码修改之前，如果有人和我一样习惯的人，可以将默认的nano编辑器改为我们熟悉的vim。对我而言，nano真是巨**难用。

1. 编辑/etc/environment文件。（这个文件用于系统全局环境变量设置,修改后重启生效）

```shell
sudo vim /etc/environment
# 文件中插入以下内容，然后保存退出
export EDITOR=/usr/bin/vim 
export VISUAL=/usr/bin/vim
```

1. 对于每用户生效,可以编辑其`~/.profile`或`~/.bash_profile`文件:

```shell
vim ~/.profile

export EDITOR=/usr/bin/vim 
export VISUAL=/usr/bin/vim

# 使其生效
source ~/.profile
```

### 正式修改密码

刚开始的`Pwd@Linux`密码是不行的，我们要换更加复杂的密码，建议用密码生成器生成一个密码，例如：[1password](https://1password.com/zh-cn/password-generator/)，可以自己自建[Vaultwarden](https://github.com/dani-garcia/vaultwarden).

1. 修改当前用户root密码，系统会提示输入旧密码,然后输入新密码两次。

```shell
passwd
```

1. 新建一个普通用户`admin`，以后我们以普通用户登陆系统，sudo安装或者执行管理员权限时，可以不输入密码。

```powershell
# 创建admin，提示中输入密码
adduser admin

# 给我的admin用户增加临时root权限，效果：sudo时不用再输入密码了
visudo
```

# 找到`#User privilege specification`,然后插入`amin ALL=(ALL) NOPASSWD: ALL` ,最后保存退出。

![c6de5da5818b97a70d466db2c0bc365c](https://img.stuarthub.us.kg/2024/04/c6de5da5818b97a70d466db2c0bc365c.png)

注意：

1. admin为例，就意味着随着本文的发布，这个用户名也会变成一个不大不小的特征，也许会被攻击者优先尝试。所以和端口一样，我强烈建议你用一个自己想到的其他用户名。
2. NOPASSWD这个设置，它的意思是vpsadmin用户临时使用root权限时，不用额外输入密码。这与一般的安全建议相反。我之所以如此推荐，是因为很多新人不顾危险坚持使用root账号就是因为用root时不用重复输入密码、觉得轻松。“两害相权取其轻”，我认为【直接用root用户的风险】大于【使用sudo时不用输密码的风险】，所以做了以上的建议。如果你希望遵守传统习惯、每次使用sudo时需要输入密码，那么这一行改成 vpsadmin ALL=(ALL:ALL) ALL 即可。
3. 在设置密码后，同第一条，一定要新开窗口，试一下能否登陆，不要把自己关外边，下面的安全设置不再重复这句话。

## 禁止root登陆

以后，我们就用普通用户登陆了，为了安全，我们就把root这头“怪兽”关在牢笼之中。

1. 禁用root登陆：文件中找到`PermitRootLogin Yes`这一项，然后把它后面的设定值改为no即可。

```shell
 sudo vim /etc/ssh/sshd_config
```

1. 重启 ssh 服务，让变更生效。

```shell
sudo service sshd restart
```

## 启用 RSA 密钥验证登录并禁止密码登陆

> 如果不喜欢密码登陆，可以该用密钥登陆，重点一点要保存好密钥，找不到就G了。

其他的常见密钥还有：

- DSA - 已经从数学层面被证明不安全，所以永远不要用它
- ECDSA - 密钥小安全性高，但其算法被指留有 NSA 的后门，如果你的 VPS 上有值得 NSA 关注的东西就不要用它
- Ed25519 - 这是一个与 ECDSA 十分类似的算法，故具有相似的性能优势。同时其文档全部公开，所以普遍认为无后门所以，如果你的设备和软件都支持的话，我建议优先选择 Ed25519 密钥。

1. 服务器创建公私钥对，输入后会在家目录下，创建一个`.ssh`的文件夹。如果是本地创建看2,3 ,4.

```shell
# ssh-keygen -t rsa -b 4096 -C "<CLIENT-NAME>"
ssh-keygen -t rsa -b 4096 -C "passwd"  # passwd可自由更换
```

2， 可以在本地MAC或者Linux本地执行上述命令，然后将公钥传上去，可上传私钥到目的机器，注意替换自己的端口，用户名和IP地址

```shell
# scp -P ssh端口 ~/.ssh/id_rsa.pub <YOURNAME>@<SERVER-IP>:/home/<YOURNAME>/.ssh/authorized_keys
scp -P 66 ~/.ssh/id_rsa.pub admin@37.23.13.93:/home/vpsadmin/.ssh/authorized_keys
```

1. 按照提示输入登录密码
2. 检查一下服务器上的.ssh目录，已经有了。
3. 【如果是在服务器创建公私钥对跳转这里】,修改 authorized_keys 文件权限为 600 （仅所有者可读可写）

```shell
chmod 600 ~/.ssh/authorized_keys
```

1. 编辑ssh配置文件,然后保存退出

```shell
sudo vim /etc/ssh/sshd_config #搜索PasswordAuthentication，把yes改成no
```

![](https://img.stuarthub.us.kg/2024/04/d5440a56dd39194f7d6b4f5010be47a9.png)

这样以来，每次登陆，我们用上传公钥，就能实现无密码登陆了 。

最后最后的提醒：

- ！！！一点不要关闭当前窗口，用公钥登陆尝试一次，查看能否登陆成功。
- ！！！一定一点要保护好自己的公私钥匙对。

# 最后的最后

建议不要开启过多端口，开启443，80 和ssh端口即可，服务可用nginx做好反向代理即可。

推荐阅读：

[

服务器自带的系统不喜欢？那就自己DD一个船新的系统吧！-我不是咕咕鸽

site.seo.description

![](https://blog.laoda.de/upload/guguge.webp)我不是咕咕鸽-VPS折腾不完全记录咕咕

![](https://blog.laoda.de/upload/guguge.webp)

](https://blog.laoda.de/archives/dd)

[

保护好你的小鸡！保姆级服务器安全教程！-我不是咕咕鸽

site.seo.description

![](https://blog.laoda.de/upload/guguge.webp)我不是咕咕鸽-VPS折腾不完全记录咕咕

![](https://blog.laoda.de/upload/guguge.webp)

](https://blog.laoda.de/archives/how-to-secure-a-linux-server)

[

【Docker系列】一个反向代理神器——Nginx Proxy Manager-我不是咕咕鸽

site.seo.description

![](https://blog.laoda.de/upload/guguge.webp)我不是咕咕鸽-VPS折腾不完全记录咕咕

![](https://blog.laoda.de/upload/guguge.webp)

](https://blog.laoda.de/archives/nginxproxymanager)
