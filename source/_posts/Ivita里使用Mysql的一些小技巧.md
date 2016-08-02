---
title: 'Ivita里使用Mysql的一些小技巧'
date: 2016-08-02 11:39:19
tags: [Mysql]
---

## 项目背景

因为旅行计划与放假安排没有完全重叠，于是向老板申请调假，在小伙伴们都放高温假的时候，在实验室干活一周。于是被临时安排到了这个叫做`Ivita`项目里，做的东西其实就是一个手环app。不过因为是临时一周，之后出现问题不好解决，所以只负责了后端的几个API实现。技术框架原本就已经定好，`slimePHP + Medoo`，数据库是`Mysql`。然后由于懒癌发作，加上很多功能`Medoo`并不能实现，于是用了不少以前没有用过的SQL语句和设置。

## Table设置
### Create和Update的自动记录时间触发

这个其实在用`Postgresql`的时候就已经用过了，结果发现`Mysql`更加简单。

~~~mysql
	created_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
~~~

### 多主键限制

`Ivita`里`uid`和`date`共同作为主键，所以需要给他们俩一起添加一个唯一性限制。

~~~mysql
	UNIQUE KEY (uid, date)
~~~

## SQL语句

### 分组求和并排序

业务需求本来求和周年月的每日数据，然后返回一个排名，不过之后这个功能没有意义了，就没有用到这个功能了。这里贴一个更好的：

~~~mysql
	SELECT orderNumber,
	       FORMAT(SUM(quantityOrdered * priceEach),2) total
	FROM orderdetails
	GROUP BY orderNumber
	ORDER BY SUM(quantityOrdered * priceEach) DESC;
~~~

### 插入重复则更新

很方便的一个功能，本来要写两条还要写代码判断的现在一句话就能解决了，不过`Ivita`用的`Mysql`版本不支持`Json`，所以不知道如果有的字段是`Json`的话，是不是依然还是那么方便。在`Ivita`里我把`Json`字段单独处理的。

~~~mysql
	insert into steps (uid, date, count, calory, kilo) values (${id}, ${value}, ${data['count']}, ${data['calory']}, ${data['kilo']}) on duplicate key update count=count+values(count), calory=calory+values(calory), kilo=kilo+values(kilo);
~~~

