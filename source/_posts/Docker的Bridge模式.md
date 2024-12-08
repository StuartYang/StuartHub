---
title: Docker的Bridge模式
mermaid: true
math: false
comments: true
hide: false
date: 2024-12-08 11:19:12
tags:
	- docker
categories:
	- VPS
---


# ****  
用户自定义bridge****

1. 在主机上通过命令`docker network ls`可以查看docker中存在的网络：

```powershell
	docker network ls
	# 输出结果：
NETWORK ID          NAME                DRIVER              SCOPE
e79b7548b225        bridge              bridge              local
666d5f1f459d        host                host                local
d0d785cf4794        none                null                local

```

1. 创建用户自定义bridge：

```bash

docker network create my-net        # 创建了一个名为"my-net"的网络
```

1. 将Web服务容器和mysql服务容器加入到"my-net"中，并观察变化：

```bash
docker network connect my-net test_demo         # 将Web服务加入my-net网络中
docker network connect my-net mysqld5.7         # 将mysql服务加入my-net网络中
```

1. 查看my-net的网络配置

```bash
# 查看my-net的网络配置
docker network inspect my-net
# 输出结果（省略部分内容）：
[
    {
        "Name": "my-net",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",      # my-net的子网
                    "Gateway": "172.18.0.1"         # my-net的网关
                }
            ]
        },
        "Containers": {
            "d6f33e9bbd60e10d02dd2eebea424a7fc129d9646c96742ec3fe467833017679": {
                "Name": "mysqld5.7",
                "EndpointID": "7d0e8d70bb523cceb4d2d8d4e3f8231fc68332c70f7f9b4e5d4abccce2b31a65",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",         # mysql服务容器的IP，与之前不同
                "IPv6Address": ""
            },
            "dd292d535d16bbfe5382e29486756f4dddfea8e9b10af769db61618d739c5c4e": {
                "Name": "test_demo",
                "EndpointID": "f7802f1af81e258f77e227609dfdcdf66c49f19776381eb8b0dca6d9e794ccad",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",         # Web服务容器的IP，与之前不同
                "IPv6Address": ""
            }
        },
    }
]

```

1. 通过容器名或别名互连： 进入到Web服务器container1中连接container2：
