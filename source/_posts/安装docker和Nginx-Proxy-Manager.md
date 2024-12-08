---
title: 安装docker和Nginx-Proxy-Manager
mermaid: true
math: false
comments: true
hide: false
date: 2024-12-08 11:23:20
tags:
	- docker
	- nginx
categories:
	- VPS
---


#   升级 packages

```shell
sudo -i # 切换到 root 用户

apt update -y  # 升级 packages

apt install wget curl sudo vim git -y  # Debian 系统比较干净，安装常用的软件
```

# 安装docker

- 海外服务器安装

```shell
wget -qO- get.docker.com | bash
```

- 国内服务器安装

```shell
curl -sSL https://get.daocloud.io/docker | sh
```

默认都自带了docker compose ，可以通过docker compose version查看。

- 设置开机自动启动

```
systemctl enable docker  # 设置开机自动启动
```

# 修改 Docker 配置（可选）

内容参考：[烧饼博客](https://u.sb/debian-install-docker/)

以下配置会增加一段自定义内网 IPv6 地址，开启容器的 IPv6 功能，以及限制日志文件大小，防止 Docker 日志塞满硬盘：

```shell
cat > /etc/docker/daemon.json <<EOF
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "20m",
        "max-file": "3"
    },
    "ipv6": true,
    "fixed-cidr-v6": "fd00:dead:beef:c0::/80",
    "experimental":true,
    "ip6tables":true
}
EOF
```

然后重启 Docker 服务：

```shell
systemctl restart docker
```

# 安装 Nginx Proxy Manager

1. 创建安装目录：

```shell
sudo -i

mkdir -p /usr/local/docker_data/npm

cd /usr/local/docker_data/npm
```

1. 创建`docker-compose.yml`

```shell
vim docker-compose.yml
```

1. Docker compose内容：
2. 1. 更加具体的配置看官网：[https://nginxproxymanager.com/guide/#quick-setup](https://nginxproxymanager.com/guide/#quick-setup)

```yaml
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

1. 检查端口是否占用，占用重新修改端口

```shell
apt install lsof  #安装 lsof

lsof -i:81  #查看 81 端口是否被占用
```

1. 运行并访问 Nginx Proxy Manager

```shell
docker-compose up -d
```

1. 我们就可以输入 [http://ip:81](http://ip:81/) 访问了。默认登陆名和密码：

```
Email:    admin@example.com
Password: changeme
```

## 更新 Nginx Proxy Manager

```shell
docker-compose down 
# 备份，防止万一
cp -r /root/data/docker_data/npm /root/data/docker_data/npm.archive 

docker-compose pull
docker-compose up -d  

docker image prune  # prune 命令用来删除不再使用的 docker 对象。删除所有未被 tag 标记和未被容器使用的镜像
```

  

---

  

推荐阅读：

- https://blog.laoda.de/archives/nginxproxymanager
