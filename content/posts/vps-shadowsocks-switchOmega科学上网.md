---
title: vps + shadowsocks + switchOmega科学上网
date: 2016-08-18 14:55:11
tags: [shadowsocks, SwitchOmega, GFW, socks5]
---

## 背景

之前用了一段时间的`Lantern`，虽然很好用，但是流量消耗得太快，跑了一天就超过了80%。正好早上登上了`budgetvm`的vps整了下数据库，所以就想顺便再把代理捣鼓一下。


![](/uploads/shadowsocks.png)

## 经过
借着`Lantern`剩下的一点流量，没费多少功夫就搜到了神器[shadowsocks](https://github.com/shadowsocks/shadowsocks)，但是很遗憾，似乎作者停止更新了。不过好在wiki还是完整的，开始按照wiki折腾了很久，不得要领。遂请教胡神，胡神很给力的给了给了一套教程[Get Through the Firewall in Scientific Method](https://github.com/sjtug/kxsw/wiki)。按照教程捣鼓了一番，还是不得要领。最后在胡神亲自带领下，终于成功翻墙，于是写下这篇笔记记录一下方法。

基本按照这个图来

    +-------------------------------------+
    |       Server A(大陆以外)             |
    |    运行shadowsocks-python服务端      |
    | 对外提供的SS IP为ip1，端口为port1      |
    +--------------+----------------------+
               |
               |
               |
    +--------------+-------------------------------------------+
    |     本机                                                 |
    | 运行shadowsocks-python客户端 + switchOmega socks5代理     |
    +----------------------------------------------------------+

本来为了方便使用，应该是在墙内设置一个Server B，然后在B上运行`shadowsocks`客户端，同时添加`cow`的二级代理，转成`http`代理，然后就方便使用了，但是测试了半天没有成功，改用上面的这种简化的方法。缺点在于本机需要运行客户端才可以开启代理。不过都设置好之后也还算方便。

## 服务器端(墙外VPS)安装和配置

首先安装`shadowsocks`，因为绑定1000以下端口和后台运行都是需要`sudo`权限的，所以需要直接在系统库中安装会比较方便一点。

```shell
    sudo pip install shadowsocks
```

安装完之后就可以直接运行`ssserver`命令了。
首先写服务端的配置文件`shadowsocks.json`：

```json
{
        "server":"▇▇.▇▇.▇▇.▇▇",
        "server_port":9000,
        "password":"FUCKXXX",
        "timeout":300,
        "method":"aes-256-cfb"
}
```

然后运行

```
    ssserver -c shadowsocks.json
```

或者后台运行，可能需要`sudo`权限

```
    ssserver -c shadowsocks.json -d start
```
这个时候服务器端的配置就完成了。

## 客户端(本机或国内VPS)配置
安装和运行方式同上，只有配置文件略有不同

```json
{
        "server":"▇▇.▇▇.▇▇.▇▇",
        "server_port":9000,
        "local_address":"127.0.0.1",
        "local_port":443,
        "password":"FUCKXXX",
        "timeout":300,
        "method":"aes-256-cfb"
}
```

这样一来，本机就有了一个ip为`127.0.0.1`，端口为`443`的`socks5`代理了。

## SwitchOmega配置
`SwitchOmega`需要先翻墙在`Chrome`商店进行安装，安装完之后添加刚才设置好的`socks5`代理即可。

但是`SwitchOmega`还需要设置自动代理，这样可以直接访问国内的网站而不用代理。打开插件的设置页面，选择`auto switch`，在`Rule list`中选择`AutoProxy`，然后在网址填入`https://raw.githubusercontent.com/gfwlist/gfwlist/master/gfwlist.txt`。点下载配置即可。

此时应该可以顺利翻墙了。

## DNS污染

这个概念我还没有搞清楚，也没有去折腾相关的解决方案，使用了胡神给的一个据说没有被污染的DNS，然后就能成功访问`Google`了。
