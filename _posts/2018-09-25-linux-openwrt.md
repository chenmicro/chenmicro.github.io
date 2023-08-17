---
title:  "Openwrt配置说明"
last_modified_at: 2023-03-02T00:00:00+00:00
categories:
  - hardware
tags: linux router openwrt
toc: true
---

路由器版本：

Model: Newifi-D2

Architecture: MediaTek MT7621 ver:1 eco:3

Firmware Version: OpenWrt 22.03.3


# 1 系统配置

## 1.1 无线中继

* 新建接口`WWAN`和`WWAN6`，分别配置为`DHCP`以及`DHCPv6`，设置防火墙区域为`wan`
  
* 在无线页面中扫描并添加，在通用设置中绑定上面的两个网络接口
  
* 当使用NAT6时，在`/etc/firewall.user`添加ip6tables规则，假设wlan设备为`wlan1`：
  
  ```shell
  ip6tables -t nat -A POSTROUTING -o wlan1 -j MASQUERADE
  ```
  
  可以参考：[Lede、PandoraBox IPv6 NAT66的实战操作【亲测】](https://www.right.com.cn/forum/thread-253712-1-1.html)

  如果子网设备的ipv6连接还是有问题，可以通过`route -A inet6`查看默认网关是否有问题，
  可能是路由距离`metric`数值太大，可通过下述指令获取网关并添加路由(注意替换脚本中的变量)。
  
  ```shell
  ip -6 route | grep default
  route -A inet6 add default gw $GW dev $DEV
  ```



## 1.2 ipv6

在上级路由没有分配PD前缀时，有三种解决方案：Relay，Passthrough和NAT6。
* Relay可以参考下列帖子：[OpenWrt 路由器如何让 lan 口主机获得 ipv6 网络访问？](https://www.zhihu.com/question/29667477/answer/93634257)，
  这种方法并不是在所有环境下都有效。
* Passthrough就是将IPV6协议的WAN与LAN桥接，会导致路由器无法获取IP地址，并不实用。
* 因此这里使用NAT6的方式，下面为设置流程。

官方文档：[OpenWrt Project: NAT6: IPv6 Masquerading Router](https://openwrt.org/docs/guide-user/network/ipv6/ipv6.nat6)

首先，要让路由器不获取ipv6前缀：

```wiki
# /etc/config/network
config interface 'wan6'
	option reqprefix 'no'
```

然后，要让lan的设备获取到ipv6地址。

1. 设置 IPv6 ULA 前缀，注意前缀的第一个数字（16进制）不要是F。

2. 配置DHCPv6

    ```wiki
    # /etc/config/dhcp
    config dhcp 'lan'
        option dhcpv6 'server'
        option ra 'server'
        option ra_default '1'
        option ra_flags 'managed-config other-config'
    ```

3. 修改防火墙规则，开启ipv6地址翻译并关闭ICMP转发

    ```shell
    # /etc/config/firewall
    config zone
          option name 'wan'
          # add
          option masq6 '1'

    config rule
          option name 'Allow-ICMPv6-Forward'
          # add
          option enabled '0'
    ```
4. 配置网络

    ```wiki
    # /etc/config/network
    config interface 'wan6'
        option sourcefilter '0'
    ```

5. 端口转发
   
   目前无法在uci中配置ipv6的DNAT，
   参考[issue#9624](https://github.com/openwrt/openwrt/issues/9624)，
   可以在防火墙配置文件中手动添加，主要需要设置`family`为`ipv6`

## 1.3 旁路由

事件的起因：由于主路由运行了一些消耗CPU的服务，在NAS中挂了PT以后，经常出现主路由连接数暴涨、CPU不堪重负的情况。为了减少主路由的负担，决定在NAS中安装Openwrt虚拟机分担主路由的一部分功能。

虚拟机的安装就不在这边说了，旁路由的相关知识可以参考下面这篇文章：

[OpenWrt中，旁路由的设置与使用 - 知乎](https://zhuanlan.zhihu.com/p/112484256)

然而，网上的文档很多是直接将旁路由作为DHCP服务器（包括上面的文章），而实际上将主路由作为DHCP服务器也是可行的。配置过程包括设置旁路由的IP以及设置主路由DHCP的推送这两个步骤：

1. 设置旁路由的静态地址：

   IPV4方面，在LAN接口页设置设备的地址（这里用`192.168.1.2`），使用和主路由相同的掩码，将网关设置为主路由的IP。

   IPV6方面，因为主路由用的是NAT，所以在全局网络选项中设置和主路由相同的ULA 前缀。然后在LAN接口页将分配长度设置成和主路由一样，并设置该设备的IPv6 后缀（这里用`::2`）。

   保存并应用后就能够看到设备获得了正确的IPV4和IPv6 地址了。

2. 设置主路由的DHCP推送

   在主路由LAN接口的DHCP服务器页面中，在高级设置中增加DHCP选项，添加自定义网关和DNS的通告：

   ```bash
   # 网关通告
   3,192.168.1.2
   # DNS通告
   6,192.168.1.2
   ```

   在IPV6设置中填写DNS服务器：`ULA::2`（**如果不设置，设备将通过主路由的ipv6解析DNS**）

   保存并应用后，重启设备的网络连接，就可以看到设备的IPV4网关、IPV4和IPV6的DNS都改为旁路由了。

DHCP选项中的第一个数字为`DHCP 选项编号`，可以直接查找相关标准资料。

按照上述步骤设置后，设备默认从旁路由解析DNS并发起连接。对于例外的设备，只要手动将其IPV4网关设置成主路由就行了。

如果哪天不想用了，只要把主路由的设置改回来就行了。



# 2 软件配置

## 2.1 HTTPS访问

1. 使用[acme](https://github.com/acmesh-official/acme.sh)申请通配符证书
2. 安装下面两个包：`luci-ssl`，`luci-app-uhttpd`
3. 将`fullchain.cer`以及`private.key`上传到路由器
4. 修改`uhttpd`配置文件`/etc/config/uhttpd.conf`，修改证书文件的路径以及监听端口
5. 重启`uhttpd`服务
6. 在防火墙中添加端口转发并重启

注意事项：

* 建议关闭`HTTP到HTTPS的重定向`，否则内网使用IP访问会有警告
* `let‘s encrypt`证书有效期为3个月，需要定期更新



## 2.2 DDNS

对于阿里云，直接使用下面仓库的脚本即可：
[h46incon/AliDDNSBash](https://github.com/h46incon/AliDDNSBash)

对于cloudflare，安装下列软件包：`luci-app-ddns`，`ddns-scripts-cloudflare`



## 2.3 Openvpn

新版的系统中设置方法又简化了。

OpenWrt官网的文档：[点我](https://openwrt.org/docs/guide-user/services/vpn/openvpn/start)

### 在路由器运行Server

**千万不要添加端口转发**

* 安装`openvpn-openssl`以及界面`luci-app-openvpn`

* 配置`/etc/config/openvpn`，现在可直接引用外部文件：

    ```wiki
    config openvpn 'custom_config'
        # 启用配置文件
        option enabled '1'
        # 使用的配置文件
        option config '/etc/openvpn/server.ovpn'
    ```
    
    配置文件例子，注意`redirect-gateway`不需要`bypass-dhcp`属性：
    
    ```wiki
    # 实现子网掩码24的子网地址扩展
    topology subnet
    # 配置为UDP协议 如果是ipv6则使用udp6
    proto udp
    # 适配器编号 如果写tun默认从0开始编号
    dev tun0
    # 服务器监听端口
    port 1194
    # 服务器地址
    server 10.8.1.0 255.255.255.0
    # 加密选项
    ncp-ciphers AES-128-CBC
    cipher AES-128-CBC
    # 允许客户端之间互相访问
    client-to-client
    # 心跳包
    keepalive 15 60
    # 超时后保留key
    persist-key
    # 超时后保留tun适配器
    persist-tun
    # 一个client证书可以同时使用多次
    duplicate-cn
    # 日志文件以及记录等级
    verb 3
    log '/etc/openvpn/openvpn.log'
    # 推送内容
    # 使客户端所有的IP请求都通过该服务器
    push "redirect-gateway def1"
    # 推送内网路由
    push 'route 192.168.1.0 255.255.255.0'
    # 推送DNS服务器
    push 'dhcp-option DNS 192.168.1.1'
    # 推送其它配置项
    push 'persist-tun'
    push 'persist-key'
    # 证书等
    <dh></dh>
    <ca></ca>
    <cert></cert>
    <key></key>
    ```
    
* 在 网络/接口 界面中添加新接口，选择openvpn的设备，不配置协议

    ```wiki
    config interface 'ovpn0'
        option proto 'none'
        # 上面服务器配置中的接口名称
        option ifname 'tun0'
    ```

* 配置/etc/config/firewall

    ```wiki
    # 通信规则
    # 允许数据包发送给OpenVPN服务器的监听端口
    config rule
        option name 'Allow-OpenVPN'
        # OpenVPN中配置的协议
        option proto 'udp'
        option src 'wan'
        # OpenVPN中的监听端口
        option dest_port '1294'
        option target 'ACCEPT'

    # 添加在lan区域中添加tun0所在的接口
    config zone
        option name 'lan'
        # add context
        list network 'ovpn0'
    ```

### 在旁路由运行Server

同样按照上述过程操作，并且在主路由中添加端口转发，不出意外的话客户端能够成功连接了。

然后你会发现只能访问Openvpn的子网和旁路由自己。
因为旁路由只有1个LAN接口，我们需要手动在路由表中添加地址转换，这里有两种方式：

1. 在自定义防火墙规则中添加（只适用于fw3）

    ```bash
    iptables -t nat -A POSTROUTING -s 10.8.1.0/24 -j MASQUERADE
    ```

2. 在`NAT Rules`中添加（同时适用于fw3和fw4）

重启防火墙，客户端测试能否访问主路由和互联网。

### 在终端运行Client

这是电脑客户端的配置

```wiki
client
# 通信协议 ivp6使用UDP6
proto udp
# 适配器类型
dev tun
# 服务器地址以及端口 地址可以使用域名
remote [ip] 1194
# 允许服务器ip变更
float
# 加密选项
ncp-ciphers AES-128-CBC
cipher AES-128-CBC
# 心跳包
keepalive 15 60
# 采用服务器校验方式
remote-cert-tls server
# 服务器为域名时始终重新解析IP
resolv-retry infinite
# 本机不绑定监听数据的端口
nobind
# 证书
<ca></ca>
<cert></cert>
<key></key>
```

### 在路由器运行Client

在OpenWrt中使用客户端文件连接远程服务器，可以参照server中的步骤，用同样的方式建立一个接口`tun0`，并设定防火墙规则为`wan`，然后将配置文件中的适配器指定为`tun0`。

如果路由器可以访问远程网络而路由器下面的设备不可以，那就是路由器的防火墙没设置好。

这时你访问某些网站时还是会出现证书错误等问题，这是因为路由器自身的DNS服务并没有被Openvpn自动修改，需要手动将上游DNS服务器改为旁路由。



## 2.4 其它模块

`wol`:

```shell
opkg install luci-app-wol
```



# 3 其它指令

查看当前连接：

```bash
cat /proc/net/nf_conntrack
```

测试openssl性能：

```bash
openssl speed -evp AES-128-GCM
```


