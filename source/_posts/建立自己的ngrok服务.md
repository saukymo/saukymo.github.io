---
title: 建立自己的ngrok服务
date: 2016-09-26 22:40:09
tags: [ngrok, DNS, go, ubuntu]
---

![](/uploads/GO_16.png)

## 需求

两个需求：

1. 组里采购了一台服务器，托管在所里的机房里，但是到目前为止还没有分配外网ip，所以需要一个将本地端口暴露在外网的解决方案；
2. 台式机是使用的所里的内网，但是本本用的路由。而且偶尔会需要离开工位，此时如果想用本本连到台式机，也需要将本地端口暴露到外网；

用过的解决方案，都是基于ngrok的：

国内的ngrok.cc，其实挺好用的，虽然速度上慢了一点。但是这次升级了版本之后，配置文件的写法没有公布出来，而且如果总是变的话也不是个长久的办法。加上这次服务器被攻击，直接关闭了三天免费服务器，这一点是无法容忍的。

原生的ngrok.io。配置比ngrok.cc要简单一点，但是服务器在美帝，更加慢了。免费版本随机出来的域名实在太难记了，而且每次都会变。并且不能绑定自己的域名，导致这个服务就今天拿来临时用了一下。

## 初步方案

自然还是用自己的服务器转发一下比较好。不过有两个问题，第一个是自己的服务器也不稳定，第二个是自己的服务器也在美帝。所以这个方案也只能是临时。但之后所里的服务器会有外网ip了，也就不需要这个服务了。其次，之后也许可以在所里的服务器上搭一个ngrok的server，这样就能以很快的速度登录到我的机器了(也许不能的原因是要绑定域名，而这个似乎很困难）

### 安装golang

~~~
sudo apt-get install build-essential
sudo apt-get install golang
sudo apt-get install mercurial
~~~

然而我的`ubuntu`版本似乎太低，安装好的go是1.0的。所以要自己安装高等级的版本，我这里安装的是1.6的版本。

~~~
sudo curl -O https://storage.googleapis.com/golang/go1.6.linux-amd64.tar.gz
sudo tar -xvf go1.6.linux-amd64.tar.gz
sudo mv go /usr/local
~~~

然后编辑profile，将go的路劲添加到系统路径里。然而我并没有成功。于是，我采用了很暴力的解决方案：

~~~
sudo ln -s /usr/local/go/bin/go /usr/bin/go
~~~

然后可以运行一下，查看是否安装成功:

~~~
go version
~~~

### 下载编译ngrok

~~~
git clone https://github.com/inconshreveable/ngrok.git ngrok
cd ngrok
~~~

接下来，是生成自认证的证书，这里我直接引用人家的一段解释：

```
Once it has finished cloning ngrok to your local machine, there is yet another thing you will need to do before you can build and compile your very own ngrok server and client and that is to create your own self-signed SSL certificate and this is required because ngrok provides the secure tunnel via TLS and in order for both the client and server to work you will need to build and compile ngrok using your self signed SSL certificate and these are linked to each other. So you cannot connect to your self-signed ngrokd server via the official ngrok client as both the server and client need to be signed using the same certificate.
```

然后是生成证书的命令，需要将`NGROK_BASE_DOMAIN`替换成自己的域名，例如我的就是"tunnel.mkdef.com"。

~~~
openssl genrsa -out base.key 2048
openssl req -new -x509 -nodes -key base.key -days 10000 -subj "/CN=[NGROK_BASE_DOMAIN]" -out base.pem
openssl genrsa -out server.key 2048
openssl req -new -key server.key -subj "/CN=[NGROK_BASE_DOMAIN]" -out server.csr
openssl x509 -req -in server.csr -CA base.pem -CAkey base.key -CAcreateserial -days 10000 -out server.crt
~~~

然后需要去托管域名的网站添加一条解析规则，例如我托管在DNSpod上了。
最后终于可以编译了！！！

~~~
cp base.pem assets/client/tls/ngrokroot.crt
make release-server release-client
~~~

生成的服务端和客户端在`bin`目录下。

## 运行测试
### 服务端

~~~
./ngrokd -tlsKey=server.key -tlsCrt=server.crt -domain="[NGROK_BASE_DOMAIN]" -httpAddr=":8080" -httpsAddr=":8081"
~~~

然后打开浏览器，访问一下你配置的网站就好了，例如“http://tunnel.mkdef.com:8080”。
看到提示说`Tunnel not found`就表示成功了。

### 客户端

首先编写配置文件`ngrok.cfg`:

~~~
server_addr: [NGROK_BASE_DOMAIN]:4443
trust_host_root_certs: false
~~~

然后执行

~~~
./ngrok -subdomain testing -config=ngrok.cfg 80
~~~

这样你就能通过 `testing.tunnel.mkdef.com:8080` 来访问你本地的80端口了。

如果是tcp连接，执行

~~~
./ngrok -config=ngrok.cfg -proto tcp 22
~~~

这是会显示出一个端口，似乎是随机的，不过好在反复执行是不变的，然后通过这个端口就能登录到你的本地机器了。没有测试多次执行是什么情况。

## TODO
目前为止只解决了部分需求，因为这个新的外网网址实在太长了，而且端口不是默认端口，所以至少还需要一步端口转发。

## Reference

[How To Install Go 1.6 on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-install-go-1-6-on-ubuntu-14-04)

[Run Ngrok on Your Own Server Using Self-Signed SSL Certificate](https://www.svenbit.com/2014/09/run-ngrok-on-your-own-server/)