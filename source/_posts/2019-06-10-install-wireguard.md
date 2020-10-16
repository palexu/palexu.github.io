---
title: ubuntu(vps)与mac(client)上安装wireguard, 并对出口流量分流
date: 2019-06-10
---

## 基本

### server, 以ubuntu为例

```bash
# 一键安装wireguard 脚本 Ubuntu
# 该脚本由 https://github.com/hongwenjun 提供, 建议自行查看该脚本内容, 了解原理
$ wget https://raw.githubusercontent.com/hongwenjun/vps_setup/master/ubuntu_wireguard_install.sh | bash
```

之后拷贝生成的 client.conf 内容

### client, 以mac为例

```bash
brew install wireguard-tools
```

## udp加速

### server

```shell
# 一键安装 udp2raw/udpspeed & kcptun
# 该脚本由 https://github.com/hongwenjun 提供, 建议自行查看该脚本内容, 了解原理
$ wget 
https://raw.githubusercontent.com/hongwenjun/WinKcp_Launcher/master/wg_udp2raw.sh | bash
```

安装完成后, 会输出如下内容:

> 请访问 https://git.io/winkcp 下载Windows客户端; https://git.io/sskcp.sh OpenWRT或KoolShar脚本
> 按以下实际信息填充    服务器IP: [服务器ip]
>   WG+SPEED+UDP2RAW 原端口: 20713 ;  UDP2RAW伪装TCP后端口: 2999 ; 转发密码: 324f83
>   SS+KCP+UDP2RAW加速: UDP2RAW伪装TCP后端口: 1999 ; SS密码: 324f83 加密协议 aes-256-gcm
>   手机SS+KCP加速方案: KCPTUN端口: 4000 ; KCP插件设置参数 mode=fast2;key=324f83;mtu=1300

### client

下载 udp2raw 与 udpspeed

https://github.com/wangyu-/udp2raw-tunnel/releases

https://github.com/wangyu-/UDPspeeder/releases

其中, 可能需要安装额外的依赖, 参见:[安装pcap和libnet](https://github.com/wangyu-/udp2raw-multiplatform/wiki/安装pcap和libnet)



放置到/etc/wireguard目录



```bash
sudo /etc/wireguard/udp2raw_mp -c -l127.0.0.1:20712 -r155.138.138.10:2999 -k 324f83 --raw-mode easy-faketcp

sudo /etc/wireguard/speederv2 -c -l127.0.0.1:20713 -r127.0.0.1:20712 -k 324f83 -f20:10 --mode 0

sudo wg-quick up wg1
```



todo: 可参考 https://github.com/atrandys/luci-udptools/blob/master/src/etc/init.d/udptools , 配置上面这两个命令的一键启动(未完成)



注意: 在wireguard中, endpoint的流量会走老的网关出去. 但是此时我们需要将endpoint指向127.0.0.1:20713(speederv2服务), 此时127.0.0.1的流量就会错误地走老的网关出去(而不是本机转发).

![image-20190611131545663](/Users/xj/code/palexu.github.io/_posts/assets/image-20190611131545663.png)

因此, 建议在 PostUp 里, 将 127.0.0.1 重新指向 -blackhole (或简单地直接移除 `route delete 127.0.0.1`)

wg1.conf

```properties
[Interface]
PrivateKey = PRIVATE_KEY
Address = 10.0.0.2/24
DNS = 8.8.8.8
MTU = 1200
PostUp = export PRIORITY=1024; source /etc/wireguard/udp-up
PreDown = export PRIORITY=1024; source /etc/wireguard/udp-down

[Peer]
PublicKey = PUBLIC_KEY
Endpoint = 127.0.0.1:20713
AllowedIPs = 0.0.0.0/0, ::0/0
PersistentKeepalive = 25
```

udp-up

```bash
#!/bin/sh
export PATH="/bin:/sbin:/usr/sbin:/usr/bin"

OLDGW=`netstat -nr | grep '^default' | grep -v 'ppp' | sed 's/default *\([0-9\.]*\) .*/\1/' | awk '{if($1){print $1}}'`

if [ ! -e /tmp/pptp_oldgw ]; then
    echo "${OLDGW}" > /tmp/pptp_oldgw
fi
echo "${OLDGW}"

dscacheutil -flushcache

route add 10.0.0.0/8 "${OLDGW}"
route delete 127.0.0.1
route add <YOU_SERVER_IP> "${OLDGW}"
```

udp-down

```bash
#!/bin/sh
#!/bin/sh
export PATH="/bin:/sbin:/usr/sbin:/usr/bin"

if [ ! -e /tmp/pptp_oldgw ]; then
        exit 0
fi

OLDGW=`cat /tmp/pptp_oldgw`

route delete 10.0.0.0/8 "${OLDGW}"
route delete 155.138.138.10 "${OLDGW}"

```

## 客户端分流

![image-20190611141859385](https://tva2.sinaimg.com/large/006tNc79ly1g3xa5q27nlj30lg0m1q41.jpg)

分流主要依靠 chnroutes 项目实现, 参考[wireguard-configuration: issue mac如何分流](https://github.com/zbinlin/wireguard-configuration/issues/1)

```shell
cd /etc/wireguard
git clone https://github.com/fivesheep/chnroutes.git
cd chnroutes
./chnroutes -p mac
```



```
PostUp = export PRIORITY=1024; source /etc/wireguard/ip-up
PreDown = export PRIORITY=1024; source /etc/wireguard/ip-down
```



### 存在问题 
### 8.8.8.8 覆盖了公司内网dns, 导致内部网站无法访问

### 是否添加了公司内部的网关

### 考虑udp2raw之后tcp阻断的替代方案


## 参考资料

1. [WireGuard wg-quick PostUp的高级玩法](https://sskaje.me/2017/06/wireguard-wg-quick-postup的高级玩法/)
2. [树莓派上安装WireGuard](https://github.com/adrianmihalko/raspberrypiwireguard)
3. [下一代VPN协议 - WireGuard](https://colin-chang.site/linux/part3/wg.html)
4. [基于chnroutes实现的Mac下的VPN国内外网络分流技术](https://blog.csdn.net/offbye/article/details/37871539)
5. [WireGuard 配置和上网流量优化](https://blog.mozcp.com/wireguard-usage/)