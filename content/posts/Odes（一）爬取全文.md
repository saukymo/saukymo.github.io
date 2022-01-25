---
title: Odes（一）爬取全文
date: 2016-06-16 10:02:25
tags: [Python, 诗经, 爬虫, lxml, postgresql]
---

## 项目想法

因为一直没有一个比较完整的项目，所以想参考[变卦](http://zhangwenli.com/biangua/)做一个小的project，展示一个内容有限但是比较有意思的东西，顺便学习实践一下前端技术，于是就选择了诗经。内容不多，一共也就305首。一共分成3个步骤吧，第一步找个网站抓一个比较完整的全文下来，整理好之后存放到数据库里，第二步写一个前端把内容展示出来，但是这边是前后端配合还是单独一个前端页面还没有想好。第三步是其他功能的加入，比如注音和释义，甚至其他的一些比如统计数据之类的功能。

时间安排上并没有计划，主要最近空闲时间比较多，又不想复习，于是才想做这么个项目。反正先把坑挖在这，什么时候能做完就只能随缘了。

## postgres安装和配置

其实这一部并不一定需要，因为内容确实不多，直接做成静态页面效果也不错。主要是为了之后第三部可能会需要比较复杂的功能时提前准备。但是没想到这一部花了比较多的时间，最后也只是能用，并没有设置成一个正常的状态。

### postgres安装

按照各类教程中的内容，只需要这一步就可以安装好数据库并且自动开启服务，端口5432。

``` sh
sudo apt-get install postgresql
```
但是执行完之后，psql并没有连上数据库，服务器上也看不到postgres的进程。

略去中间大量的搜索和尝试的过程，在`/usr/lib/postgresql/9.1/bin/`目录里找到常用的命令，于是按照手动启动数据库的方式启动：

```sh
/usr/lib/postgresql/9.1/bin/postgres -D ~/data
```

这里提示我不能用root权限开启服务，恍然大悟，之前没有成功启动的原因可能就是因为我一直使用的root用户(使用的服务器虽然用了很长一段时间了，但是当时仅仅安装了一个wordpress就没有登上去过了。所以一直都是root用户。)但是它为什么没有任何提示呢？！于是得到一个教训，新服务器第一件事就是建立一个新的非root用户。建了一个新的用户`saukymo`之后，终于成功开启了服务。

之后的过程由于大量试错，现在不能准确回忆起来了，不过应该还是通过这个目录下的`initdb`和`createdb`成功建立了一个数据库`odes`。

在Mac中，可以通过`brew`安装，安装完成之后就可以使用`psql`了，之后也能安装`psycopg2`的python库了。

```sh
brew install postgresql
```

直接使用`psql`命令进入数据库，然后给用户设置密码：

```sh
\password saukymo
```
### Postgres允许远程连接
默认情况下，只允许本机连接数据库，如果需要远程连接到数据库，需要设置postgres允许远程连接。设置比较简单，首先修改`data`目录下的`pg_hba.conf`文件，加入一行

```
host    all             all             0.0.0.0/0               md5
```
这样就能允许所有的ip通过密码访问数据库了。

然后修改`postgresl.conf`文件，设置`listen_addresses`为任意即可，即：

```
listen_addresses = '*'
```

然后重启服务，我这里因为安装方式不太正确，命令为：

```sh
/usr/lib/postgresql/9.1/bin/pg_ctl -D ~/data restart
```

此时就能在其他机器上连接上数据库了，完整的`psql`命令为：

```sh
psql -U saukymo -h ▇▇.▇▇.▇▇.▇▇ -d odes -W
```
按照提示输入之前设置的密码即可。

### 安装psycopg2
这个库是python用来连接`postgres`数据库的，通过`pip`安装即可

```sh
pip install psycopg2
```

需要注意的是，这个库并不能兼容`Pypy`，如果需要和`Pypy`一起工作的话，需要安装[psycopg2cffi](https://pypi.python.org/pypi/psycopg2cffi)，简单设置之后，就可以兼容了，而且原来的代码不需要变化。

可以用以下代码进行测试：

```python
#coding: utf-8
import psycopg2 as pg

if __name__ == '__main__':
    db = pg.connect(database="odes", user="saukymo", password="▇▇▇▇▇▇▇▇", host="▇▇.▇▇.▇▇.▇▇", port="5432") 
```

## 爬取诗经全文

需要的库比较少，除了`psycopg2`，另外就是`lxml`用来解析HTML，其他的库暂时都不需要了。安装方法都是通过`pip`。

目标网页一开始选择的是[诗歌库](http://www.shigeku.com/xlib/gd/sj305/)，因为发现结构挺简单的，好像也比较完整和规范。所以还比较顺利地抓完了目录和第一篇正文，但是开始全文抓取的时候，第二篇就编码错误了。本来也没多想，编码错误再正常不过了，于是尝试了多个不同的中文编码，全部报错，`ISO`虽然能解析完，但是解析出来的都是看不懂的蚂蚁文。

看了一下网页编码，`GB2312`，应该没有问题呀，但是解析不对。安装了`chardet`探测，发现出错的页面里，虽然主要是'gb2312'，但是置信度确实不高，反复检查之后发现，原来是原文中本来就存在解析不了乱码，显示成了方框。

思考了一番，没有什么好的方法，只好换抓另一个网页：[中文百科在线](http://www.zwbk.org/MyLemmaShow.aspx?lid=76385)，简单看了一下，还不错，目录简单，结构还比较规范，于是就开始抓了，结果发现所谓的规范，仅仅是看上去而已。

### 爬取目录

目录爬取比较简单，全部代码如下：

```python
url = "http://www.zwbk.org/MyLemmaShow.aspx?lid=76385"
connect = urlopen(url)

content = connect.read()
page = html.parse(StringIO(content.decode('utf-8')))
table = page.xpath("//table/tr/td[2]/div/div[7]")

collect_list = []
for links in table[0].find_class("classic"):
	title = links.text_content().split(u"·")
	if len(title) > 3:
		page_url = links.attrib.get("href")
		collect_list.append({
			"category": title[1],
			"collect": title[2],
			"title": title[3],
			"page_url": page_url
			})
```

因为网站给这些目录全部加上了`classic`类，所以先用`xpath`找到具体的代码块，然后在其中找`classic`类即可。

然后通过map来批量抓取全部正文：

```python
result = map(lambda x: get_fulltext(**x), collect_list)
```

### 爬取正文

正文部分就比较麻烦了，首先内容不是被单独封装在一起的，而是被各种`h2`,`br`等标签分隔开了，所以提取比较麻烦，正好同学之前问了相同的问题，当时没有给出好的解决方法，这次搜索了一下发现，`lxml`提供了`tail`方法，可以得到两个子元素之间的内容，于是比较好的解决了这个问题。代码如下：

```python
def get_fulltext(category, collect, title, page_url):
	print category, collect, title
	try:
		content = urlopen(page_url).read()
	except Exception as e:
		print e
		return {"title":"%s_%s_%s" % (category, collect, title), "text":""}
		
	page = html.parse(StringIO(content.decode('utf-8')))
	table = page.xpath('//td[2]/div/div[7]')
	if len(table) > 0:
		odes = table[0]
		skip_state = 0
		full_text = []
		for element in odes.getchildren():
			text = ""
			if element.tag is etree.Comment:
				continue
			if element.tag == "h2":
				skip_state += 1
			if (element.tag == "div") and skip_state == 1:
				text = element.text_content().strip()
			if (element.tag == "br") and skip_state == 1:
				if element.tail is not None:
					text = element.tail.strip()
			if text != "":
				full_text.append(text)
	return {"title":"%s_%s_%s" % (category, collect, title), "text":full_text}
```
大意是提取第一个`h2`和第二个`h2`之间的所有内容。比较顺利的得到了大部分正文。有如下几个问题和我采用的解决方法(好像全都是手动完善的...)

	1. 鹊巢这一篇抓不到正文内容，原因不详，不过只有这一篇，所以手工补全；
	2. 有的正文混入了注释，手工删除；	
	3. 抓一部分之后，会出现连续的503错误，猜测网站做了防抓取或者平时访问没有这么频繁，短时间内挂掉了。多抓几次，手工拼合。
	4. 这个网站，颂部分少了很多篇，比如清庙之什应该是10篇的，它只有一篇，所以手工从诗歌库上复制粘贴过来，然后替换成相同格式。
	5. 有的篇章没有换行，或者有多余空格，手工清除。
	6. 保存的文本用python读取不出来，原因不详。用`Sublime Text`打开，复制粘贴到新的文件，一切正常。

秉承的唯一信念就是，反正只有300篇，也是一次性的工作，做完就可以了。所以手工修复了所有的bug，目前看上去效果良好。

## 上传数据库

### 建表
一共就一个表，代码如下：

```mysql
	CREATE TABLE IF NOT EXISTS odes (
        id SERIAL PRIMARY KEY,
        p_class TEXT,
        p_group TEXT,
        p_subgroup TEXT,
        title TEXT,
        full_text TEXT,
        create_time TIMESTAMP,
        update_time TIMESTAMP 
```
其中，`p_class`是指风雅颂三类，`p_group`是细分，比如`周南`,`召南`。`p_subgroup`指的是什。其他看名字就知道了。
另外 `SERIAL`是自增序列，`postgres`会自动新建一个自增的序列作为它的值。当然也可以手工建立，然后链接过来。

### 设置触发器
然后为创建和更新时间设置触发器自动更新。

```mysql
	CREATE OR REPLACE FUNCTION auto_timestamp() RETURNS trigger AS $auto_timestamp$
        BEGIN
        IF (TG_OP = 'INSERT') THEN
                NEW.create_time := current_timestamp;
           NEW.update_time := current_timestamp;
        ELSIF (TG_OP = 'UPDATE') THEN
          NEW.update_time := current_timestamp;
        END IF;
            RETURN NEW;
        END;
        $auto_timestamp$ LANGUAGE plpgsql;

    DROP TRIGGER IF EXISTS auto_timestamp ON odes;
    CREATE TRIGGER auto_timestamp BEFORE INSERT OR UPDATE ON odes
        FOR EACH ROW EXECUTE PROCEDURE auto_timestamp();
```

### 上传
具体解析就不写了，因为保存的格式是我自定义的。上传部分的代码如下：

```python
def upload_one_ode(metadata):
    script = """
        INSERT INTO odes (p_class, p_group, p_subgroup, title, full_text) 
            VALUES (%(p_class)s, %(p_group)s, %(p_subgroup)s, %(title)s, %(full_text)s);
    """
    cu.execute(script, metadata)
```

最后记得`commit`就行。

```python
db.commit()
```

### 在数据库中查看数据

首先，连接到数据库中，看一下有多少条记录。

```
odes=# select count(*) from odes;
 count
-------
   305
(1 row)
```

然后随便抽一条数据检查一下：

```
odes=# select * from odes where id=180;
 id  | p_class | p_group | p_subgroup | title |                          full_text                           |        create_time         |        update_time
-----+---------+---------+------------+-------+--------------------------------------------------------------+----------------------------+----------------------------
 180 | 雅      | 小雅     | 彤弓之什    | 吉日   | 吉日维戊，既伯既祷。田车既好，四牡孔阜。升彼大阜，从其群丑。+| 2016-06-15 21:53:41.366594 | 2016-06-15 21:53:41.366594
     |         |         |            |       | 吉日庚午，既差我马。兽之所同，麀鹿麌麌。漆沮之从，天子之所。+|                            |
     |         |         |            |       | 瞻彼中原，其祁孔有。儦儦俟俟，或群或友。悉率左右，以燕天子。+|                            |
     |         |         |            |       | 既张我弓，既挟我矢。发彼小豝，殪此大兕。以御宾客，且以酌醴。+|                            |
     |         |         |            |       |                                                              |                            |
(1 row)
```

其中`+`应该是换行符，可以看到还是比较规整的。

## 小结
本来是一个非常简单的任务，还是花了不少时间和精力，而且过程中很多问题都只是绕了过去，并没有真正解决。不过好在最后的结果还是很不错的。目前还有一个问题值得商榷，就是保存正文时，换行是我手工换的，之后显示出来不一定好看，所以是否可以考虑不保存换行符，之后动态的调整。不过这个问题之后还是修改的空间，可以之后再根据情况调整。

全文可以在[这里](https://github.com/saukymo/odes/blob/master/crawler/full_text.txt)下载到