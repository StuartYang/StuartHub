---
title: OpenWrt定时备份服务器
mermaid: true
math: false
comments: true
hide: false
date: 2024-12-08 11:27:32
tags:
	- rsync
categories:
	- OpenWrt
---




# 安装 &配置 OpenSSH

默认 OpenWrt 软件源中包含 OpenSSH，可以通过以下命令安装：
```bash

opkg update
opkg install openssh-server

```

如果需要客户端工具（例如 ssh 和 ssh-keygen），可以额外安装：
```bash

opkg install openssh-client

```

安装完成后，需要对 OpenSSH 进行初步配置。
```bash

vim /etc/ssh/sshd_config

```

关键配置选项：

```plaintxt

" 修改默认端口（可选，提高安全性）：
Port 22
" 允许 root 登录(根据需求设置，但不推荐）
PermitRootLogin yes/no
" 禁用密码登录（推荐使用密钥认证），如果不在乎可以为设置为yes，进行密码登陆，因为我的软路由在家，
" 设置为yes
PasswordAuthentication no

```

如果是首次安装 OpenSSH，需要生成主机密钥：
```bash

ssh-keygen -A

```

启动**OpenSSH**并设置为开机自启：

```bash

/etc/init.d/sshd start

/etc/init.d/sshd enable

```

```bash
-- 检查服务状态：
/etc/init.d/sshd status
-- 使用客户端连接到 OpenWrt 设备：
ssh username@<设备IP地址>

```


**重要‼️ ：在执行完`ssh-keygen -A`后，会在`~/ssh`目录下生成公私钥对，将公钥复制到要目标机器（备份谁，谁是目标机器）的`~/.ssh/authorized_keys`内。（安全考虑，后续的脚本里登陆是密钥登陆，所以未配置密码）**

# 安装&配置rsync

> rsync 是一个高效的文件同步工具，支持增量传输，可以通过 SSH 拉取远程文件夹。

- **Linux/macOS**：通常已预装。

-  **Windows**：可以通过 WSL进行安装

【注意】
- 为了防止因为版本的问题导致无法同步，建议使用`apt update`(ubuntu Linux)或者`opkg update`(Openwrt)进行软件源头的更新。

- Mac建议使用`HomeBrew`进行安装和维护，Mac自己带的rsync不要删除，在`~/.zshrc`或者`~/.profie`中进行替换系统指令即可，配置如下：

```bash

 # 打印出rsync安装的目录位置 ，我默认安装在/opt/homebrew/Cellar/rsync/3.3.0下
 # 用于下文替换
 brew info rsync 
 
```

`~/.zshrc`文件中追加：

```plaintxt
RSYNC_HOME=/opt/homebrew/Cellar/rsync/3.3.0
export PATH=$RSYNC_HOME/bin:$PATH
```

```bash

# 让配置生效
source ~/.zshrc

# 验证rsync是否生效，当前时间是2024年，mac出场自带的是rsync2.x的版本，如果输出是3.x，说明自己安装的rsync生效
rsync --version
```


## 创建 rsync 拉取脚本

创建一个脚本文件，比如 `sync_data.sh`：

```shell

#!/bin/bash

# 远程服务器信息
SERVER="user@server"
# 以下内容自行修改
REMOTE_DIR="/path/to/remote/folder"
LOCAL_DIR="/path/to/local/folder"

# 使用 rsync 拉取文件
rsync -avz --delete --progress $SERVER:$REMOTE_DIR $LOCAL_DIR

```

• -a：归档模式，保留权限和时间戳。

• -v：显示详细信息。

• -z：压缩传输。

• --delete：同步时删除本地多余的文件（可选，根据需求使用）。

• --progress：显示传输进度。


赋予执行权限：
```bash
chmod +x sync_data.sh
```


在终端运行脚本，确保可以成功同步：
```bash
./sync_data.sh
```

如果一切正常，本地文件夹会被拉取并更新。


# crontab配置

> crontab 是 Linux 和 Unix 系统中用于管理和定时执行任务的工具。它的核心功能是让系统按照指定的时间计划自动运行任务脚本或命令。通过 crontab，可以实现自动化操作，如备份文件、定期清理日志、定时同步数据等。

这里我们用crontab做定时任务，定期去备份目标机器的数据。

1. 设置定时任务，你需要编辑 crontab 文件。每个用户都有自己的 crontab 文件，使用 crontab 命令来编辑它：

```bash

crontab -e

```

这将打开当前用户的 crontab 配置文件。如果是第一次使用，系统会让你选择一个编辑器（例如 nano、vi 等）。


2. 在 crontab 文件中，你可以添加一行来指定任务的执行时间和要执行的脚本。

```plantxt

# backup task，这里表示每两天执行一次备份脚本，cron表达式自行网上查找
0 14 */2 * * /path/to/local/folder/sync_files.sh

```

3. 完成编辑后，保存文件并退出编辑器。
4. 使用以下命令查看当前用户的 crontab 任务
```bash
crontab -l
```


# 总结

以上配置就可以，进行定期对远程服务器进行增量拉取同步
- OpenSSH：密码/密钥远程登陆
- rsync：增量同步
- crontab ： cron表达式定期执行rsync脚本


# 题外话

在 OpenWrt 上设置 Samba（SMB 文件共享），可以实现局域网文件共享功能。以下是具体步骤：

1. 登录到 OpenWrt 的 SSH 终端。
2. 更新软件包列表,安装 Samba：
```bash

opkg update
# samba4-server 是 Samba 服务本体。
# luci-app-samba4 提供了通过 OpenWrt Web 界面配置 Samba 的功能。
opkg install samba4-server luci-app-samba4
```

3. 创建被共享的目录，如果有忽略
4. 为共享目录设置适当的权限：
```bash

chmod -R 777 /me/samba/share

```

5. 配置Samba 服务，这里有两种方式：一种是界面，一种是命令行，这里采用命令行，界面的配置方式Google上有很多。

```bash
# 打开samba4的配置文件
vim /etc/config/samba4
```

在文件中追加以下内容：
```plaintxt

config sambashare
    option name 'share' 
    option path '/mnt/samba/share'
    option read_only 'no'
    option guest_ok 'yes'
    option create_mask '0666'
    option dir_mask '0777'
```

	• option name：共享名,smb访问的路径后缀
	• option path：共享目录路径。
	• option guest_ok：设置为 yes 允许匿名访问,因为是局域网访问，我就设置成可以匿名访问，如果是公网，建议新建系统用户进行授权登陆（详细看附加功能）

6. 保存并退出，启动Samba服务,设置开机自启动
```bash

/etc/init.d/samba4 start

/etc/init.d/samba4 enable
```

7.  在客户端访问共享
	 - **Windows**：打开文件资源管理器，输入地址：\\\\<OpenWrt_IP>\share（例如：\\\\192.168.1.1\share）。 
	 - **Mac/Linux**： 打开文件管理器，选择 **连接到服务器**，输入地址：smb://<OpenWrt_IP>/share。

****附加功能**（可选）

如果不想让共享文件夹允许匿名访问，可以设置密码保护：

1.  为用户设置 Samba 密码
```bash

# 系统会提示输入密码两次。完成后，用户将被添加到 Samba 用户数据库。
sudo smbpasswd -a <sambauser>
# 默认情况下，新添加的 Samba 用户是禁用的。使用以下命令启用该用户
sudo smbpasswd -e <sambauser>

```
2. 为samba用户设置目录` /srv/samba/share`的共享权限 （按需配置）
```shell
# 允许 sambauser 读写访问。
sudo chown sambauser:sambauser /srv/samba/share
# 其他用户无访问权限
sudo chmod 770 /srv/samba/share


# 如果多个用户共享一个目录，可创建一个用户组，将共享目录权限赋给组：
sudo groupadd sambashare
sudo usermod -aG sambashare sambauser
sudo chown -R :sambashare /srv/samba/share
sudo chmod 770 /srv/samba/share

```

3. 编辑 Samba 配置文件`/etc/samba/smb.conf`：
```ini

config sambashare
    option name 'share' 
    option path '/mnt/samba/share'
    option read_only 'no'
    option guest_ok 'no'  ; 竞争匿名访问 
    option create_mask '0666'
    option dir_mask '0777'

```

4. 重启Samba服务
```bash

sudo systemctl restart smbd

```

5. 测试连接，参看上文

## 附加调试

```bash
# 检查 Samba 配置是否正确
testparm
# 查看 Samba 日志
sudo tail -f /var/log/samba/log.smbd

```
