---
title: Hysteria配置
mermaid: true
math: false
comments: true
hide: false
date: 2024-12-08 11:18:02
tags:
	- vps
categories:
	- VPS
---

  
  

# 系统

- **Ubuntu 22.04 64 Bit**

---

# **服务器测评脚本**

```
 curl -L <https://gitlab.com/spiritysdx/za/-/raw/main/ecs.sh> -o ecs.sh && chmod +x ecs.sh && bash ecs.sh -m 1
```

# **相关链接**

- 参看链接：[https://bulianglin.com/archives/hysteria2.html](https://bulianglin.com/archives/hysteria2.html)
- Hysteria 2下载：[https://github.com/apernet/hysteria/releases](https://github.com/apernet/hysteria/releases)
- Hysteria 2文档：[https://v2.hysteria.network/zh/](https://v2.hysteria.network/zh/)
- Android客户端（SFA）：[https://install.appcenter.ms/users/nekohasekai/apps/sfa/distribution_groups/publictest](https://install.appcenter.ms/users/nekohasekai/apps/sfa/distribution_groups/publictest)
- IOS客户端（TestFlight）：[https://testflight.apple.com/join/AcqO44FH](https://testflight.apple.com/join/AcqO44FH) （1.5.0 beta版支持Hysteria 2）
- IOS客户端（AppStore）：[https://apps.apple.com/us/app/sing-box/id6451272673](https://apps.apple.com/us/app/sing-box/id6451272673) （暂不支持Hysteria 2）

# **服务器步骤**

```bash
 #一键安装Hysteria2
 bash <(curl -fsSL <https://get.hy2.sh/>)
 
 #生成自签证书
 openssl req -x509 -nodes -newkey ec:<(openssl ecparam -name prime256v1) -keyout /etc/hysteria/server.key -out /etc/hysteria/server.crt -subj "/CN=bing.com" -days 36500 && sudo chown hysteria /etc/hysteria/server.key && sudo chown hysteria /etc/hysteria/server.crt
 
 #启动Hysteria2
 systemctl start hysteria-server.service
 #重启Hysteria2
 systemctl restart hysteria-server.service
 #查看Hysteria2状态
 systemctl status hysteria-server.service
 #停止Hysteria2
 systemctl stop hysteria-server.service
 #设置开机自启
 systemctl enable hysteria-server.service
 #查看日志
 journalctl -u hysteria-server.service
 
```

bash

```bash
 cat << EOF > /etc/hysteria/config.yaml
 listen: :443 #监听端口
 
 #使用CA证书
 #acme:
 #  domains:
 #    - a.com #你的域名，需要先解析到服务器ip
 #  email: test@sharklasers.com
 
 #使用自签证书
 tls:
    cert: /etc/hysteria/server.crt
    key: /etc/hysteria/server.key
 
 auth:
   type: password
   password: passwd #设置认证密码

 masquerade:
   type: proxy
   proxy:
     url: <https://bing.com> #伪装网址
     rewriteHost: true
 EOF
```

bash

步骤：

1. 一键安装Hysteria2
2. 设置开机自启
3. 启动Hysteria2
4. 生成自签证书
5. 设置证书
6. 防火墙开发443/80端口
7. 重启Hysteria2
8. 查看日志

## **客户端配置**

v2ray:

```yaml
 server: ip:443
 auth: 123456
 # 流控，不要太大
 bandwidth:
   up: 20 mbps
   down: 100 mbps
 
 tls:
   sni: bing.com
   insecure: true #使用ca时需要改成true
 
 socks5:
   listen: 127.0.0.1:1080
 http:
   listen: 127.0.0.1:8080
```

yaml

sing-box配置文件(Android/IOS)

```json
{
  "dns": {
    "servers": [
      {
        "tag": "cf",
        "address": "https://1.1.1.1/dns-query"
      },
      {
        "tag": "local",
        "address": "223.5.5.5",
        "detour": "direct"
      },
      {
        "tag": "block",
        "address": "rcode://success"
      }
    ],
    "rules": [
      {
        "geosite": "category-ads-all",
        "server": "block",
        "disable_cache": true
      },
      {
        "outbound": "any",
        "server": "local"
      },
      {
        "geosite": "cn",
        "server": "local"
      }
    ],
    "strategy": "ipv4_only"
  },
  "inbounds": [
    {
      "type": "tun",
      "inet4_address": "172.19.0.1/30",
      "auto_route": true,
      "strict_route": false,
      "sniff": true
    }
  ],
  "outbounds": [
	   {
       "type": "hysteria2",
       "tag": "proxy",
       "server": "127.0.0.1",   // 改为自己VPS的IP地址
       "server_port": 443,
       "up_mbps": 50,   // 上行流孔
       "down_mbps": 100, // 下行流控
       "password": "passwd", // 密码
       "tls": {
         "enabled": true, //开启tls
         "server_name": "bing.com", // 伪装域名
         "insecure": true
       }
    },
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "block",
      "tag": "block"
    },
    {
      "type": "dns",
      "tag": "dns-out"
    }
  ],
  "route": {
    "rules": [
      {
        "protocol": "dns",
        "outbound": "dns-out"
      },
      {
        "geosite": "cn",
        "geoip": [
          "private",
          "cn"
        ],
        "outbound": "direct" 
      },
      {
        "geosite": "category-ads-all",
        "outbound": "block"
      }
    ],
    "auto_detect_interface": true
  }
}
```

json


