---
title: Hexo和Github搭建Blog
date: 2016-04-17 10:19:42
tags: hexo
---

![](/uploads/github_hexo.png)
## Hexo

[Hexo](https://hexo.io/zh-cn/)是一个很简洁好用的静态Blog框架，第一次看到是在[始终](http://liam0205.me/)的博客，当时觉得主题很漂亮，速度也还不错，在页面底下看到了`Hexo`，于是就去看了一下介绍，结果发现好简单啊。加上去年写了一学期`Latex`，觉得`Markdown`肯定只会更简单，于是就按照这个[教程](http://wsgzao.github.io/post/hexo-guide/)也搭了一个，于是这篇日志基本就参(zhao)考(chao)这篇教程写了。

## 准备工作
教程用的windows，我用的Mac，其实差不多，Mac还简单一些，正好给了我个机会证明这篇日志~~是原创~~不全是照抄的。

### Homebrew

Mac下缺的东西都可以用[Homwbrew](http://brew.sh/)来装，很方便。打开就是安装命令。

### Node.js
~~~sh
$ brew install npm
~~~

### hexo
然后就可以安装`hexo`了

~~~sh
$ npm install hexo-cli -g
$ npm install hexo --save
~~~

## Hexo配置

### 初始化
执行以下命令

~~~sh
# 安装 Hexo 完成后，请执行下列命令，Hexo 将会在指定文件夹中新建所需要的文件。
$ hexo init <folder>
$ cd <folder>
$ npm install

# 新建完成后，指定文件夹的目录如下
.
├── _config.yml
├── package.json
├── scaffolds
├── scripts
├── source
|      ├── _drafts
|      └── _posts
└── themes
~~~

### 安装插件
这个我是按照教程来的，但是有几个明显用不到的就可以不装了，比如git之外的几个`deployer`， 还有好几个`generator`。

~~~sh
$ npm install hexo-generator-index --save
$ npm install hexo-generator-archive --save
$ npm install hexo-generator-category --save
$ npm install hexo-generator-tag --save
$ npm install hexo-server --save
$ npm install hexo-deployer-git --save
$ npm install hexo-deployer-heroku --save
$ npm install hexo-deployer-rsync --save
$ npm install hexo-deployer-openshift --save
$ npm install hexo-renderer-marked@0.2 --save
$ npm install hexo-renderer-stylus@0.2 --save
$ npm install hexo-generator-feed@1 --save
$ npm install hexo-generator-sitemap@1 --save
~~~

### 预览
~~~sh
$ hexo g #生成public静态文件
$ hexo s #本地发布预览效果
INFO  Start processing
INFO  Hexo is running at http://0.0.0.0:4000/. Press Ctrl+C to stop.
~~~
本地做的所有更改都会实时的更新，可以随时看到刚刚所做的修改，非常方便，平时也可以另外开一个进程一直跑，然后再修改。

### 新文章和新页面
~~~
hexo new "文章标题"
hexo new page "关于"
~~~
这两个命令其实都是生成的`Markdown`文件，如果想要修改文章或者编辑页面，编辑相应的`.md`文件就可以了。

## Github配置和自动部署
[Github Pages](https://pages.github.com/)是一个Github开发的给用户托管静态网页的功能，自由度和稳定性都非常高，十分良心。

### github
1. 新建项目，名字必须和用户名一致，格式为`username.github.io`；
2. 稍等几分钟之后就能访问静态主页了`http://username.github.io`；
3. 编辑`_config.yml`

	~~~
	deploy:
	  type: git
	  repo: <repository url>
	  branch: [branch]
	  message: [message]
	~~~
4. 命令`hexo d`就会自动把`public`里的文件全部同步到github上，并且访问主页就能看到所做的修改。

### 域名设置

1. 在`public`目录下创建`CNAME`文件，里面填写自己的域名；
2. 给自己的域名添加CNAME记录，推荐[DNSPod](https://www.dnspod.cn/)，配置页有详细的介绍；
3. 将`CNAME`文件同步到Github上，很快就能通过自己的域名访问网站了。

## Hexo主题设置
目前用的是[Next](http://theme-next.iissnan.com/getting-started.html#activate-next-theme)，链接有详细的安装和设置教程。

参考其中头像的设置，可以用类似的方法设置favicon，平时在文章中插入图像的链接也是一样的。

## 关于Markdown编辑器

其实我一直用`Sublime Text`写代码，我相信它也一定能够写`Markdown`，不过我估计它不支持所见即所得，所以我也没有去配置它。

目前我用的是`MacDown`，基本我能想到的它都支持，包括代码高亮，而且效果基本和Hexo生成的一样，目前看来是我用过的最好的`Markdown`编辑器，唯一的遗憾在于不支持文档管理，不过这个功能有没有都有一定道理。

目前发现的和`Hexo`的区别有：

1. `Hexo`文件前面有一块配置区域`Macdown`解析不了；
2. 插入图片的文件路径不一致；
3. \#开头的标题要空格，不然`Hexo`不会解析成标题，而`Macdown`没有这个问题。
4. 父标题和子标题的\#数目要连续，不然生成的目录可能会有bug。
5. 使用`MathJax`的时候，\_前面要加`\`，不然解析不正确

## 使用MathJax

参考[Next的官方文档](http://theme-next.iissnan.com/third-party-services.html#others)

编辑 主题配置文件， 将 `Mathjax` 下的 enable 设定为 true 即可。 cdn 用于指定 `MathJax` 的脚本地址，默认是 `MathJax` 官方提供的 CDN 地址。

~~~sh
# MathJax Support
mathjax:
  enable: true
  cdn: //cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML
~~~

## 备份源文件

来源：[Tips for Hexo configuration](http://www.yuthon.com/2016/04/11/Tips-for-Hexo-configuration/)
之前采用新建一个repo的方法备份文件，然后发现使用`hexo d`的时候，会部署所有的文件，然后也没有找到办法恢复。

其实，只需要新建一个分支就可以备份源文件了

~~~sh
$ git init
$ git checkout -b source
$ git add .
$ git commit -m "init"
$ git remote add origin git@github.com:saukymo/saukymo.github.io.git
$ git push origin source
~~~

## 部署后自动备份源文件
来源: [自动备份Hexo博客源文件](http://notes.wanghao.work/2015-07-06-%E8%87%AA%E5%8A%A8%E5%A4%87%E4%BB%BDHexo%E5%8D%9A%E5%AE%A2%E6%BA%90%E6%96%87%E4%BB%B6.html)

安装`shelljs`

~~~sh
$ npm install --save shelljs
~~~

在`Hexo`根目录的`scripts`文件夹下新建一个js文件，文件名随意取， 如果没有`scripts`目录，请新建一个。

然后在脚本中，写入以下内容：

~~~js
require('shelljs/global');

try {
	hexo.on('deployAfter', function() { //当deploy完成后执行备份
		run();
	});
} catch (e) {
	console.log("产生了一个错误<(￣3￣)> !，错误详情为：" + e.toString());
}

function run() {
	if (!which('git')) {
		echo('Sorry, this script requires git');
		exit(1);
	} else {
		echo("======================Auto Backup Begin===========================");
		cd('~/Documents/mkdef');    //此处修改为Hexo根目录路径
		if (exec('git checkout source').code !== 0){
			echo('Error: Git checkout branch source failed');
			exit(1);
		}
		if (exec('git add --all').code !== 0) {
			echo('Error: Git add failed');
			exit(1);

		}
		if (exec('git commit -am "From auto backup script\'s commit"').code !== 0) {
			echo('Error: Git commit failed');
			exit(1);

		}
		if (exec('git push origin source').code !== 0) {
			echo('Error: Git push failed');
			exit(1);

		}
		echo("==================Auto Backup Complete============================")
	}
}
~~~

注意修改为自己的远程仓库名和相应的分支名。

此时，运行`hexo d`后的结果如下：

~~~sh
$ hexo d
INFO  Deploying: git
INFO  Clearing .deploy_git folder...
INFO  Copying files from public folder...
[master d747a0d] Site updated: 2016-05-29 01:22:33
 5 files changed, 7 insertions(+), 13 deletions(-)
To git@github.com:saukymo/saukymo.github.io.git
   bb4cdbd..d747a0d  HEAD -> master
Branch master set up to track remote branch master from git@github.com:saukymo/saukymo.github.io.git.
INFO  Deploy done: git
======================Auto Backup Begin===========================
Already on 'source'
M	"source/_posts/Hexo\345\222\214Github\346\220\255\345\273\272Blog.md"
M	themes/next
[source 1786b36] From auto backup script's commit
 1 file changed, 5 insertions(+)
To git@github.com:saukymo/saukymo.github.io.git
   49ab33a..1786b36  source -> source
==================Auto Backup Complete============================
~~~

## 压缩静态资源
安装[hexo-all-minifier](https://github.com/unhealthy/hexo-all-minifier)

~~~sh
$ npm install hexo-all-minifier --save
~~~

然后 `hexo g`时就会自动压缩静态资源



