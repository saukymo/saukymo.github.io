---
title: Odes（二）页面设计与实现
date: 2016-06-20 13:46:52
tags: [Python, html, bootstrap, Javascript, Jquery, CSS, 诗经]
---

## 整体结构

网站按照之前在Everstring常用的前后端结构，后端只提供api，前端一个html从后端请求数据然后展示，用nginx讲两者的流量区分开来。本来前端需要`gulp`配置一通流水线工具的，但是折腾了一天也没有弄出个样子来，干脆放弃这些高端玩意，先把东西用最原始最简单的方法做出来，有需求了才有动力换新的方法。

## 后端实现

由于目前只有一个展示诗文正文的一个页面，所以api非常简单。一开始的版本如下：

```python
@app.route("/odes-api/<int:id>")
def get_ode_by_id(id):
	script = """
		WITH s AS (SELECT * FROM odes where id = %s) SELECT row_to_json(s) FROM s;
	"""
	cu.execute(script, (id,))
	res = cu.fetchall()
	if res is None:
		raise ValueError
	else:
		return jsonify(res[0])
```

请求一个id的数据，把对应诗文的内容和相关的一些数据全部传输过去。这里错误处理暂时没有实现。

后来为了实现上一篇和下一篇的功能，考虑了一番决定还是通过这一个api把相邻两个诗文的数据一并传输过去，对应的sql语句为：

```mysql
WITH s AS (SELECT * FROM odes where id in (%s, %s, %s)) SELECT array_to_json(array_agg(row_to_json(s))) FROM s;
```

这样一来，问题就在于前端如何知道哪个数据是自己的，哪个数据是相邻页面的，于是手工调整顺序，将请求的数据放在中间。于是丑陋的代码实现如下：

```python
	if (id == 1):
		res[0][0], res[0][1], res[0][2] = res[0][2], res[0][0], res[0][1]
	if (id == MAXID):
		res[0][0], res[0][1], res[0][2] = res[0][1], res[0][2], res[0][0]

```

但是这样会出现几个问题，第一个是顺序是由postgres默认决定的，程序没有做校验，也没有办法校验（各个数据一致，没有字段标识区别开），正确结果依赖于各个部分默契合作，这样显然是不合理的。第二个是传输查询的内容过多，然而很多信息是毫无必要的，减慢了页面的加载速度。所以重新设计如下：第一，不采用循环的方式，第一篇没有上一篇，最后一篇没有下一篇(一方面处理逻辑要简单很多，更主要的是这个功能确实无所谓)。第二，分两次查询，因为查询是在服务器本地进行的，查询两次代价并不大，第二次仅仅查询相邻篇章的标题。代码如下：

```mysql
	WITH s AS (SELECT title FROM odes where id in (%s, %s)) SELECT array_to_json(array_agg(row_to_json(s))) FROM s;
```

然后把结果插入到请求的数据中去：

``` python
res[0]["pre_title"] = navi[0][0].get('title') if id > 1 else None
res[0]["next_title"] = navi[0][-1].get('title') if id < MAXID else None
```

其中`res`是第一次查询的结果，`navi`是第二次查询的结果。这样就显式的声明了前后文的标题，也就不会出现顺序的错误了，而且大大减少了传输的数据。

## 前端实现

目前只实现了原文的展示功能，整个页面分为如下几个部分：

	1. 顶部导航条，导航条目前有两个部分：左上角的Brand和右上角的随机浏览按钮。之后左上角还想加上搜索框。
	2. 上下翻页栏。
	3. 目录结构显示，这个功能实现了，但是不知道放在哪里比较好看，所以被我注释掉了。
	4. 标题和正文。

### 顶部导航条

html代码如下：

```html
	<nav class="navbar navbar-default navbar-fixed-top">
        <div class="container">
            <a class="navbar-brand" href="#"><img id="brand" src="favicon.png"></a> 
            <ul class="nav navbar-nav navbar-right">
                <li>
                    <a id="random_ode" href="#" class="btn-lg">
                        <span class="glyphicon glyphicon-random" aria-hidden="true"></span>
                    </a>
                </li>
            </ul>
        </div>
    </nav>
```

就是bootstrap的默认组件而已，稍微值得一提的是随机数的产生：

```javascript
function getRandomInt(x) {
  return Math.floor(Math.random() * x) + 1;
}
```

应该是可以产生1到x之间的任意整数的，包括x。

### 上下翻页
就是两个简单的超链接，一个推到左上角，一个推到右上角。左右的篇章id通过api请求得到，这个之前已经提到过。

```html
	<div id="navigate">
        <span class='pull-left'>
            <a id="left_ode" class="navigate">
            </a>
        </span>
        <span class="pull-right">
            <a id="right_ode" class="navigate">
            </a>
        </span>
    </div>
```

### 正文部分
正文部分分为标题和内容两个部分，值得一提的`css`部分。

```html
	<div id="ode">
        <div id="title"></div>     
        <div id="fulltext"></div>
    </div>
```

由于我觉得诗经这种长短参差不齐的诗句，全部左对齐会好看一些，但是又希望标题能够居中，并且和正文对齐。所以需要它的宽度和内容自适应。解决方法是将它设置为`inline-block`。
另外，在上传时，内容中就包含了换行符，以此来保证诗文显示出来的格式正确(为了限制长度，手工调整过一次，不过调整得非常不仔细，所以可能会出现内容连贯，但是被我强行换行的情况。)，所以需要能够自动显示出换行符。解决方法是`pre`。
于是，完整的css设置如下：

```css
#ode{
	text-align: center;
	font-family: Helvetica, Tahoma, Arial, STXihei, "华文细黑", "Microsoft YaHei", "微软雅黑", SimSun, "宋体", Heiti, "黑体", sans-serif;
}

#title{
	font-size: 3.5em;
	margin: 5% auto;
}

#fulltext{
	font-size: 2.5em;
	margin: auto;
	text-align: left;
	white-space:pre;
	display:inline-block; 
	*display:inline; 
	*zoom:1; 
}
```

## 部分JavaScript功能代码
### 获取页面参数
由于浏览器得到的只是一个html文件，所以它得不到请求的参数，需要从`url`手工解析出来。解析的`javascript`代码如下：

```javascript
var getUrlParameter = function getUrlParameter(sParam) {
    var sPageURL = decodeURIComponent(window.location.search.substring(1)),
        sURLVariables = sPageURL.split('&'),
        sParameterName,
        i;

    for (i = 0; i < sURLVariables.length; i++) {
        sParameterName = sURLVariables[i].split('=');

        if (sParameterName[0] === sParam) {
            return sParameterName[1] === undefined ? true : sParameterName[1];
        }
    }
};

page_id = getUrlParameter('id');
if (!page_id) {
	page_id = 1;
}
```

下面的一小段代码是如果没有解析到参数，默认显示第一篇，这样就可以通过最简单的域名访问到网站了：[odes.mkdef.com](http://odes.mkdef.com)

### 字符串替换
由于`lodash`使用的时候需要一些额外的工作，为了简单抄了一段实现字符串替换功能的代码，效果不错。

```javascript
String.prototype.replaceAll = function(search, replacement) {
    var target = this;
    return target.replace(new RegExp(search, 'g'), replacement);
};
```

## 小结
我写前端的经验真的非常匮乏，所以这些实现仅仅保证了看起来是设想的那样子，至于是不是最优，会不会有其他潜在的缺点就不得而知了，至少目前还没有暴露出来。这篇文章仅仅当作给自己的一份前端笔记好了。