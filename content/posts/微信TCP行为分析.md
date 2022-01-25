---
title: 微信TCP行为分析
date: 2016-05-19 14:36:48
tags: [tcp, wireshark, wechat, Plotly, Python]
---

#### 环境设置

实验环境：

1. iphone6 微信 WeChat6.3.16
2. macbook pro 分享wifi，网卡en0
3. wireshark 抓包，并存为Wireshark/tcpdump/...- pcap格式
4. tapo在osx中编译失败，所以采用上传数据到另一台ubuntu服务器中分析

测试功能：

1. 聊天功能： 发送文字， 发送图片， 发送小视频；接收文字，接收图片，接收小视频。
2. 朋友圈：发图文状态，发小视频；接收图文状态，接收小视频。

抓包的结果记录如下：

|功能 | 发送者ip和端口 | 接收者ip和端口 | TCP选项 | 文件名|
|-------------| -------------------- | -------------------- | ------------------| -------------------------|
|发送文字|192.168.2.2.53671|223.167.105.117.443|mss 1460,nop,wscale 5,nop,nop,TS val 459787696 ecr 0,sackOK,eol|wechat\_send\_text|
|发送图片|192.168.2.2.53673|125.39.132.125.443|mss 1460,nop,wscale 5,nop,nop,TS val 458540380 ecr 0,sackOK,eol|wechat\_send\_picture|
|发送小视频|192.168.2.2.53680|125.39.171.17.443|mss 1460,nop,wscale 5,nop,nop,TS val 458953493 ecr 0,sackOK,eol|wechat\_send\_video|
|接收文字|223.167.105.117.443|192.168.2.2.53671|mss 1460,nop,wscale 5,nop,nop,TS val 459286337 ecr 0,sackOK,eol|wechat\_receive\_text|
|接收图片|125.39.132.125.443|192.168.2.2.53698|mss 1460,nop,wscale 5,nop,nop,TS val 459286337 ecr 0,sackOK,eol|wechat\_receive\_picture|
|接收小视频|125.39.171.17.443|192.168.2.2.53764|mss 1460,nop,wscale 5,nop,nop,TS val 460156224 ecr 0,sackOK,eol|wechat\_receive\_video|
|朋友圈发图文|192.168.2.2.65283|123.151.79.111.8080|mss 1460,nop,wscale 5,nop,nop,TS val 896590244 ecr 0,sackOK,eol|friend\_cycle\_picture|
|朋友圈发视频|192.168.2.2.65293|123.151.79.111.8080|mss 1460,nop,wscale 5,nop,nop,TS val 896497923 ecr 0,sackOK,eol|friend\_cycle\_video|
|朋友圈收图文|123.126.121.175.http|192.168.2.2.54055|mss 1460,nop,wscale 5,nop,nop,TS val 896835785 ecr 0,sackOK,eol|friend\_cycle\_download\_picture|
|朋友圈收视频|124.161.14.78.http|192.168.2.2.65378|mss 1460,nop,wscale 5,nop,nop,TS val 896835785 ecr 0,sackOK,eol|friend\_cycle\_download\_video|

#### tcpdump过滤脚本
由于tapo分析时需要设置服务器ip，所以这里暂时只过滤出于测试机器相关的记录即可，测试机器的ip为192.168.2.2

``` sh
for f in `ls pcap | grep pcap`
do
    name=${f%.*}
    echo $name;
    tcpdump -r pcap/$name.pcap host 192.168.2.2 > txt/$name.info
    tcpdump -r $i host 192.168.2.2 -w filter/$i.pcap
done
```

遍历所有pcap文件，用tcpdump过滤出与测试机器192.168.2.2有关的记录，存两份，一份以可阅读的形式存在txt目录下以便手工分析，一份仍然以pcap的格式存在filter目录下，用tapo分析。

#### tapo分析
根据上面抓包分析出的服务器ip和端口，生成分析脚本：

```sh
./tcp_tool  -f pcap/friend_cycle_download_picture.pcap -s 123.126.121.175 -p 80 -t down > rst/friend_cycle_download_picture.txt
./tcp_tool  -f pcap/friend_cycle_download_video.pcap -s 124.161.14.78 -p 80 -t down > rst/friend_cycle_download_video.txt
./tcp_tool  -f pcap/friend_cycle_picture.pcap -s 123.151.79.111 -p 8080 -t up > rst/friend_cycle_picture.txt
./tcp_tool  -f pcap/friend_cycle_video.pcap -s 123.151.79.111 -p 8080 -t up > rst/friend_cycle_video.txt
./tcp_tool  -f pcap/wechat_receive_picture.pcap -s 125.39.132.125 -p 443 -t down > rst/wechat_receive_picture.txt
./tcp_tool  -f pcap/wechat_receive_text.pcap -s 223.167.105.117 -p 443 -t down > rst/wechat_receive_text.txt
./tcp_tool  -f pcap/wechat_receive_video.pcap -s 125.39.171.17 -p 443 -t down > rst/wechat_receive_video.txt
./tcp_tool  -f pcap/wechat_send_picture.pcap -s 125.39.132.125 -p 443 -t up > rst/wechat_send_picture.txt
./tcp_tool  -f pcap/wechat_send_text.pcap -s 223.167.105.117 -p 443 -t up > rst/wechat_send_text.txt
./tcp_tool  -f pcap/wechat_send_video.pcap -s 125.39.171.17 -p 443 -t up > rst/wechat_send_video.txt
```

这里有一个小问题在于，发送和接收文本的pcap文件分析的结果为空，手工查看之后发现这两个功能的pcap文件与其他文件没有什么不用，只是采用了ssl协议加密，但是按照我的理解这应该不影响包序列的分析，所以原因不明。

#### 作图
使用tapo分析过后的结果作为作图的数据，需要注意的是，当文件中包含多个tcp链接时，tapo的结果会重新标记时间。这里采用累加的方法，将多个tcp连接绘制在同一张图上，所以真实的时间可能会缩短(多个tcp连接中间重叠)或延长(两个tcp连接中间有间隔)。

还有需要考虑的一点是，对于丢包的情况，tapo的结果中表现为乱序的sequence number，在图上表示就是一个下凹的曲线，我觉得这里应该采用接收端ack的序列作图更加合理。

作图采用plotly的python库, 这里只列出朋友圈发视频的sequence图和inflight_size图为例：
![](/uploads/friend_cycle_post_video_sequence_number.png)
![](/uploads/friend_cycle_post_video_inflight_size.png)

其他的图可以在pic文件下找到。

#### 结果分析

对于收发文字这个功能，客户端与服务器只使用了一个tcp连接，当退出当前聊天窗口时会发送一个keep-alive，回到窗口时会继续采用这个tcp连接发送。这也可能是导致tapo不能正确的分析抓包数据的原因(没有捕获到tcp连接的建立和结束)

对于收发图片来说，是很明显的一张图片建立一个tcp连接；而小视频出现了一个tcp连接发送了两个小视频的情况。

文字、图片、视频都是不同的服务器，而朋友圈是一个单独的服务器(不管是图片还是视频)。

其他参数记录如下：

| 功能 | 丢包率 | 乱序率 | 
| ----| ------| -----| 
| 发送图片| 0.000000 | | 
| 发送小视频|0.000000 || 
| 接收图片|  | 0.164835| 
| 接收小视频| | 0.112540 | 
| 朋友圈发图文|0.000000  || 
| 朋友圈发视频|0.004049 || 
| 朋友圈收图文| | 0.027027| 
| 朋友圈收视频| |0.040092| 



