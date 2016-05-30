---
title: pypy加速抓取Sina后复权数据
date: 2016-04-14 14:07:36
tags: [pypy, Python, multiprocessing, 爬虫]
---

## pypy
[PyPy](http://pypy.org/)是一个独立的解析器， 通过即时编译(JIT,Just-in-time)代码避免逐行解释执行来提升运行速度。我们一般使用的Python一般是使用C实现的,所以一般又叫CPython。PyPy采用python实现，速度最快可以达到CPython的10倍左右。

PyPy对纯Python的模块支持的非常好，支持的模块可以在[这里](http://packages.pypy.org/)看到。但是PyPy对C模块的支持还不是很好，主要是对numpy的支持完成度还不够高，所以常用的一些科学运算库也就都不兼容PyPy了。所以个人感觉PyPy主要是应用在服务器和爬虫上。

## multiprocessing.dummy
`multiprocessing.dummy`和`multiprocessing`是两个执行并行任务的库，其中前者是多线程库，后者是多进程库，但是具有相同的api，所以可以很方便的在多线程和多进程之间切换。

由于GIL的原因，Python的多线程其实是单线程交替执行的，所以对于CPU密集的任务来说，多线程其实并不会有很好的效果。但是对于IO密集型的任务，多线程实现简单轻量也有很好的加速效果，值得一试。

举一个简单的爬取网页的例子：

~~~python
import urllib2 
from multiprocessing.dummy import Pool as ThreadPool 

urls = []

pool = ThreadPool(4) 
results = pool.map(urllib2.urlopen, urls)

pool.close() 
pool.join()
~~~

这个例子采用了4个线程，通过`pool.map()`来分发任务，结果依次保存在`results`中，其中`pool.join()`是等待所有线程结束之后再执行后续的代码。

## tushare
[TuShare](http://tushare.org/)是一个免费、开源的python财经数据接口包。它的数据也是从网上各种数据源抓取整理过来的，并且采用统一的结构，返回pandas的dataframe。并且它整合了通联的数据，所以这个包的数据数量和质量都是很不错的。

我的任务是采集A股市场的复权数据，tushare本来可以轻松完成这个任务，但是速度特别慢，
2800支个股4个线程采集需要将近50分钟的时间，对于一个需要每天运行一次的程序来说时间有点长。通过查看源代码，发现其实它是抓取的[sina的复权数据页面](http://vip.stock.finance.sina.com.cn/corp/go.php/vMS_FuQuanMarketHistory/stockid/600900.phtml?year=2016&jidu=1)，而且是每支股票单独抓取的，所以它一共调了将近3K次api来请求整个html页面，然后从中解析出数据。(其实是6K次，因为新浪那个页面是按季度来查询的，然后当时正好跨越了第一和第二两个季度)

于是自然想试试用PyPy来加速了，加上之前也没有用过PyPy，正好通过这个机会来测试一下PyPy的效果。

## PyPy的下载和安装
首先在官网的[下载](http://pypy.org/download.html)页面下载最新版本的PyPy：

~~~
wget https://bitbucket.org/pypy/pypy/downloads/pypy-5.0.1-linux64.tar.bz2
~~~

然后解压放到任意位置，并用PyPy的作为virtualenv的解释器。

~~~
virtualenv -p /path/to/pypy/bin/pypy env
source env/bin/activate
~~~

此时运行python就能看到PyPy的信息了：

~~~
Python 2.7.10 (bbd45126bc69, Mar 18 2016, 21:35:08)
[PyPy 5.0.1 with GCC 4.8.4] on linux2
Type "help", "copyright", "credits" or "license" for more information.
~~~

此时就可以通过`pip`来安装各种需要用到的第三方包了，这里主要是用到`lxml`，通过这个包可以很容易的解析html页面，从而提取出表格中的数据。

~~~python
 from io import StringIO
 from lxml import html
 from urllib2 import urlopen, Request
 
 def get_h_data(share, quarters):
      data = {}
      for y, q in quarters:
          url = "http://vip.stock.finance.sina.com.cn/corp/go.php/vMS_FuQuanMarketHistory/stockid/%s.phtml?year=%d&jidu=%d"%(share.get('code'), y, q)
          request = Request(url)
          text = urlopen(request, timeout=10).read()
          text = text.decode('GBK')
          page = html.parse(StringIO(text))
          table = page.xpath('//*[@id=\"FundHoldSharesTable\"]/tr')

          for tr in table[1:]:
              date = tr[0].text_content().strip()
                  data[date] = tr[3].text_content()

      share["data"] = data
      return share
~~~

可以看到，通过`lxml`来解析html页面是非常方便的，这个地方由于Sina本身的表格是没有`<tbody>`标签的，所以只能通过`<tr>`标签来提取数据，然后把第一行表格头去掉，由于我只需要收盘价，所以直接取的`tr[3]`的内容。

事实上，如果这里使用`pandas`，可以通过`read_html()`来直接将表格转换成`pandas.DataFrame`。但是由于PyPy不支持`pandas`，所以这里的数据只能手工提取出来。

## Exception and Retry
由于这里需要通过网络请求第三方的数据，所以为了避免因为各种意外情况导致的错误，我们需要用`try`把请求的部分包起来，使得即使个别数据出现了错误或者某一次请求意外地没有成功时，我们能够继续请求其他的数据或者重复请求失败的数据。

之所以把这一块单独写出来，是因为一开始自己写了一个很丑陋的`Retry`过程，当时就觉得这么简单的功能应该有更加优雅的实现，结果很快就看到了一个不错的实现方式，所以这里把这个更好的方法写下来。

~~~python
for _ in range(3):
	try:
		# Do some thing 
		break
	except Exception as e:
		print e
		
	else:
		# Continue do something
		pass
~~~
这样就可以很简单实现重复3次的功能了！

## functools.partial
这里`get_h_data`含有两个参数，但是`pool.map()`只能传一个参数，当然我们也能很简单的将并行的函数改写成一个参数的新函数，但是我们可以通过`functools.partial`来更加优雅的封装它。

~~~python
import functools

quart_get_h_data = functools.partial(get_h_data, quarters=quarters)
~~~

然后`quart_get_h_data()`就可以当做只有一个参数的函数来使用了！

~~~python
pool = ThreadPool(4)
result = pool.map(quart_get_h_data, share_list)
pool.close()
pool.join()
~~~

使用PyPy运行的程序抓取时间只需要5分钟！是之前的10倍。当然由于程序实现的方法也不一样，所以这个10倍并不准确，但是由于时(lan)间(ai)原(fa)因(zuo)，我也不去比较PyPy和Cython的效率差别了。

## 总结
这次的爬虫很好的解决了爬数据的效率问题，可以看到PyPy安装使用简单，效率超级高，虽然使用限制较大，但是对于第三方依赖较小的程序，PyPy绝对是一个很好的选择。

目前PyPy的开发团队主要的工作就是在支持`numpy`，如果能够完美的支持`numpy`，那么Python的速度问题也就能得到很好的解决，以后用Python做科学计算也就更加方便了。
