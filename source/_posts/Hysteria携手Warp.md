---
title: Hysteria携手Warp
mermaid: true
math: false
comments: true
hide: false
date: 2024-12-08 11:26:39
tags:
	- vps
categories:
	- VPS
---


#  背景

众所周知，RackNerd 家的机器胜在便宜，带宽和流量的便宜的就像不要钱一样。线路中规中矩吧，都这么便宜了，也不能要求什么（你不嫌我穷，我也不嫌你垃）。

一直好好用了半年，最近出现了几个问题：首先，登陆网站一直跳谷歌验证码（对，就那个消防栓和公交车，啊啊啊 我人麻了）；其次，因为我有RSS阅读的习惯，订阅里面有一些Yb上的视频就没法在不登陆的状态下看了，这让我很头疼；最后就是，OpenAI无法使用（好吧，我更加喜欢用Gemini）。

面对这样的问题，该怎么办呢，我从网上找了一圈，大部分说的是IP欺诈值过高，（IP检测网址：[https://ip-score.com/](https://ip-score.com/)）。可是问题得去解决啊，我看很多帖子说，可以找VPS提供商跟换IP，或者重新购买新的VPS。这两种方案，我都不适合，重新买，我穷啊，其次我手头已经有两台VPS了，属实没必要。换IP，估计RackNerd不会给我换的（毕竟都用了多半年了，回本是回本了 哈哈）。后来，我找了这个方案—Hysteria 2 + CloudFlare Warp+。

# 思路

好了，我废话说的太多了，正式介绍一下思路：因为上述描述的问题出现在出站流量上，所以我们只需要将Hysteria的流量中转给本地的Warp上，Warp己收到的请求会将请求发给Cloudflare边缘节点，接下来就是边缘节点代替我们进行请求服务。这就将我们的节点“洗白”。

  

# 配置

  

## Warp的安装&配置

  

> 作者的github地址（可以去点一个starred）: [https://github.com/fscarmen/warp-sh](https://github.com/fscarmen/warp-sh)

### 安装

  

脚本：

```shell
wget -N https://gitlab.com/fscarmen/warp/-/raw/main/menu.sh && bash menu.sh [option] [lisence/url/token]
```

直接复制运行就ok，一路默认，注意在选择运行模式，选择非全局模式。如果设置了全局模式，入口的流量都回被warp接管，这不是我们想看到的，因为可能VPS还部署着我们的服务，会导致服务无法访问。

如果不小心设置成了全局模式，可以通过`warp g` 命令将全局模式切换成非全局。

其他的warp命令如下：

```shell

 再次运行用 warp [option] [lisence]，如 

 warp h (帮助菜单）
 warp n (获取 WARP IP)
 warp o (临时warp开关)
 warp u (卸载 WARP 网络接口和 Socks5 Client)
 warp b (升级内核、开启BBR及DD)
 warp a (更换账户为 Free，WARP+ 或 Teams)
 warp p (刷WARP+流量)
 warp v (同步脚本至最新版本)
 warp r (WARP Linux Client 开关)
 warp 4/6 (WARP IPv4/IPv6 单栈)
 warp d (WARP 双栈)
 warp c (安装 WARP Linux Client，开启 Socks5 代理模式)
 warp l (安装 WARP Linux Client，开启 WARP 模式)
 warp i (更换支持 Netflix 的IP)
 warp e (安装 Iptables + dnsmasq + ipset 解决方案)
 warp w (安装 WireProxy 解决方案)
 warp y (WireProxy socks5 开关)
 warp k (切换 wireguard 内核 / wireguard-go-reserved)
 warp g (切换 warp 全局 / 非全局)
 warp s 4/6/d (优先级: IPv4 / IPv6 / VPS default)
```

  

至于代理模式是选warp，warp+还是team模式。大家可以看cloudflare的官网介绍，按需选择。我这里选择了默认的warp，因为我没有过多安全上的考虑，同时该脚本回就近找寻cf的节点，对我而言是够用的。

## 安装WARP Linux Client

  

命令行输入`warp c`， 我们把socket的端口设置为1080，后续使用，如果忘记设置，安装完毕后，命令行输入`warp -h` , 接下来输入按照提示，输入3，修改为1080。

### 结尾

这样我们Warp就安装完了，其他的配置说明请参考作者github。接下来配置Hysteria2。

## Hysteria的出站配置

目前，Hysteria 支持以下几种出站类型：

- `direct`：通过本地网络直接连接。
- `socks5`：SOCKS5 代理。
- `http`：HTTP/HTTPS 代理。

由于HTTP/HTTPS 代理在协议层面不支持 UDP。直接又不是我们想要的，所以选择`socks5`。

在Hysteria的yaml配置文件增加以下内容：

```yaml
outbounds:
  - name: my_outbound # 名字随便填
    type: socks5 # 类型 socks5
    socks5:
      addr: 127.0.0.1:1080  # socket的地址：端口
      # socks5的用户名密码，因为在本机代理，所以这里不需要填写，删除以两行也行
      # username: username  
      # password: password 
```

注意配置放在`sniff 和 asquerade` 之间，就是要按配置的顺序书写配置文件，按理说yaml是不限制这些的，可能和协议本身读取顺序关系，或者是我自己搞错缩进，无法连上服务器。保险一点，就是按顺序书写配置。

然后重启服务就可以。

### hy2常用命令

- 设置开机启动 `systemctl enable hysteria-server`
- 启动 Xray `systemctl start hysteria-server`
- 关闭 Xray `systemctl stop hysteria-server`
- 重启 Xray `systemctl restart hysteria-server`
- 检查运行状况 `systemctl status hysteria-server`

### 结尾

开始愉快的玩耍啦！我的观察，虽然又加了层warp，但是延迟不会多出很多，因为warp回就近寻找cf的边缘节点。当然了，本来延迟就170ms多，多个10ms也没什么了，哈哈哈，不影响使用。再见👋
