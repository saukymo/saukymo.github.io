---
title: odes前端改造
date: 2017-05-07 22:20:52
tags: [Bulma, Responsiveness, breadcrumb]
---

## Bulma
Bulma是一个轻量级的前端框架，仅仅包含CSS文件，而且有很多特别酷炫的[主题](https://jenil.github.io/bulmaswatch/)。第一次看到之后就想用它来写一个前端页面，正好Odes的页面功能比较简单，而且之前实现的时候有很多地方都没有做好，正好借此机会做一个小的改造。

### 替换掉Bootstrap
因为之前用bootstrap其实也只是为了用它的样式而已，主要功能还是得通过jQuery完成，于是用Bulma之后，就可以毫不犹豫地干掉Bootstrap了。

直接在页面内插入[css链接](https://cdnjs.com/libraries/bulma)，然后将bootstrap的样式替换成bulma的样式就可以了。

Bootstrap是包含部分icon的，odes里也有用到，于是还需要添加[font-awesome的css](https://cdnjs.com/libraries/font-awesome)才行。

## 添加面包屑导航
之前的api里其实已经写好了类别的请求、更新的代码，不过因为样式一直调不好的原因，没有显示在界面上。

Bulma最近才刚刚把这个加入到[feature list](https://github.com/jgthms/bulma/issues/298)里, 所以只能自己实现。

实现很简单，抄自[](https://codepen.io/superpikar/pen/JWjZOE), html代码如下
```
<ul class="breadcrumb">
	<li></li>
</ul>
```

CSS代码

```css
ul.breadcrumb {
    outline-style: none;
    display: block;
    padding: 0.75rem;
    padding-left: 0;
    background: #fff;
}

ul.breadcrumb li {
    display: inline-block;
}

ul.breadcrumb li.is-active {
    color: #848484;
}

ul.breadcrumb>li+li:before {
    padding: 0 5px;
    color: #848484;
    content: "/\00a0";
}

ul.breadcrumb li a:hover {
    text-decoration: underline;
}
```

和原版相比，就去掉了边框阴影而已。


## 设备宽度
之前在手机上看，一直是网页原版等比例缩小，以为是手机浏览器没有设置正确。然后今天才发现，原来是要在页面设置不同设备采用不同的页面宽度：

```
<meta name="viewport" content="width=device-width, initial-scale=1">
```

加上之后，页面就能变成移动版的宽度了，如果页面实现正确的话，页面效果要比之前好很多。

使用Chrome调试的时候，Console上面有一个按钮可以很方便的切换桌面版和移动版，还能选择适配不同的设备，非常方便。

## 自动换行
为了保证诗能够保留之前的段落间隔，所以展示的时候要保留换行符，之前的设置是：
```
pre {
	white-space: pre
}
```
这样有一个问题就是，当宽度超过页面宽度的时候，尤其是在手机上，会显示不完全。 后来发现原来有一种设置可以很方便的做到既保留原有的换行符，在长度过长的时候，又会自动换行：

```
pre {
	white-space: pre-wrap;
	word-wrap: break-word;
}
```

## 待完成
目前有两个计划。 一个是添加上目录索引，这个比较简单。 一个是添加搜索，这个调研了一下，也不是特别复杂。 争取下周能完成。

再就是希望能有注音和释义，这两个比较麻烦，首先是文本来源上，爬取、存储、展示感觉都不是那么简单，需要再好好考虑考虑。

记录一个特别赞的网站：[](http://graphemica.com/)，里面包含所有的字符，常用字有释义，中文有拼音，有各类编码等等等等，特别走心，界面也很好看。

