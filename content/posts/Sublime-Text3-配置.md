---
title: Sublime Text3 配置
date: 2016-05-30 15:37:48
tags: [Sublime Text3, Python, Markdown]
---

## Sublime Text3 介绍
[Sublime Text3](https://www.sublimetext.com/)是一款轻量级、跨平台的、拓展功能丰富的编辑器，使用python编写，自然也是用来写python的不二选择。然而，通过丰富强大的插件配置，ST也可以用于其他各种语言的编写。这里主要介绍我常使用的python和`Markdown`的配置。

## Package Control
要想在ST上使用插件，最方便的方法就是通过`Package Control`来安装，不过我们首先得手工安装它本身。

安装文档可以参考官方的[主页](https://packagecontrol.io/installation)，使用``Ctrl+` ``调出console，复制以下代码

```
import urllib.request,os,hashlib; h = '2915d1851351e5ee549c20394736b442' + '8bc59f460fa1548d1514676163dafc88'; pf = 'Package Control.sublime-package'; ipp = sublime.installed_packages_path(); urllib.request.install_opener( urllib.request.build_opener( urllib.request.ProxyHandler()) ); by = urllib.request.urlopen( 'http://packagecontrol.io/' + pf.replace(' ', '%20')).read(); dh = hashlib.sha256(by).hexdigest(); print('Error validating download (got %s instead of %s), please try manual install' % (dh, h)) if dh != h else open(os.path.join( ipp, pf), 'wb' ).write(by)
```

其实是压成一行的python代码，不过我还是不把它高亮了。安装完成之后，就可以通过`Ctrl+shift+P`来打开命令面板，然后在里面输入`install`，就能找到`install package`的选项，回车之后就能看到所有的插件列表了。

## 主题美化
作为一个看脸的世界，肯定首先要把界面得漂漂亮亮的才会有动力继续用下。虽然说，ST本身的界面已经非常好看了，我自己也对着这个朴素的界面用了很多很多年。但是今天在[dribbble](https://dribbble.com/search?q=sublime)上看到了一个崭新的世界。不得不承认，程序员和设计师的审美差距还是有一定距离的。

最后采用的这个[Material theme](http://equinusocio.github.io/material-theme/)主题，并且添加了所有可选选项(我相信人家的审美每一个细节都比我做得好)。效果非常好，python的高亮看着略不舒服，换成`OceanicNext`会好很多。

根据项目主页上的图片来看，`Markdown`的配色应该是这个主题设置的，然而我在第一时间并没有看到效果，倒是反反复复折腾了很久的`Markdown`插件之后才突然正常了，不过只要设置正常了，效果非常非常赞。

## Python插件配置

这里提到的插件，都是通过`Package Control`里直接输入名字就能安装的。

### [SideBarFolders](https://github.com/titoBouzout/SideBarFolders)
如果同时开发好几个项目时，ST打开项目文件夹比较麻烦，有了这个插件之后，菜单上就会多出一项`Folders`，里面就可以方便切换最近打开的目录，非常的方便。同时，如果文件夹位置发生了变动，导致之前的目录失效了，它也会提醒删除，也算是一个不错的处理方式(要求不要太高)。

### [SublimeREPL](https://github.com/wuub/SublimeREPL)
之前很长一段时间里，在ST跑python程序都不能中止，让我很是郁闷，前几年从来没有这个问题啊。后来发现是因为有一次强迫症发作，只挑了自己认识的插件安装，结果落下了这个插件。事实上，正是这个插件默默的提供了一个内置的解释器来运行python程序，也只有这样，才能通过快捷键中止程序。目前看来，对于终端的交互支持得不是很好，不过平时开发的时候，也不会用到终端输入了吧..

### [SideBarEnhancements](https://github.com/titoBouzout/SideBarEnhancements)
侧边文件栏功能加强的插件，虽然说实话我并没有仔细对比过多了哪些功能，因为之前的功能我也用得不多。不过直观上感觉右键菜单确实长了好多，之后会多多关注一下。

### [SublimeCodeIntel](https://github.com/SublimeCodeIntel/SublimeCodeIntel)
支持很多语言的自动补全提醒，再也不用dir一个一个看了。提示非常高大上，不过如果记错了函数位置，然后找半天找不到还是比较尴尬的(怪我咯)

### [Virtuallenv](https://github.com/AdrianLC/sublime-text-virtualenv)
可以很方便的激活虚拟环境，事实上我也就用过这一个功能..其他管理的功能应该也都有，但是没有用过。不过发现这个插件和`SublimeREPL`不能一起使用，也就是说用那个插件不能按照我目前的习惯使用`virtualenv`，用这个插件的话就不能中止程序了。Anyway, 这个坑还得多试试，目前看来这个插件估计用得不会太多。

### [SublimeGit](https://github.com/SublimeGit/SublimeGit)
洛洛推荐的Git集成，大部分时候都非常好用，除了本地merge出现冲突的时候出现过Bug，当然也有可能是打开方式不太对。之后自己的项目基本也都用Github管理了，所以这个插件应该会用得更多。

## Markdown插件配置
### [Markdown Extended](https://github.com/jonschlinkert/sublime-markdown-extended)
安装完之后，Syntax里会多出一个类型`Markdown Extended`，将`.md`的文件和这个新类型绑定起来之后，就能看到丰富的配色了。其实我不确定到底是这个插件的作用还是之前那个`Material theme`的作用才有了我现在的`Markdown`配色，总之很赞。

### [OmniMarkupPreviewer](https://github.com/timonwong/OmniMarkupPreviewer)
用来预览`Markdown`文件的，其实如果我只是为了写`Hexo`的话，直接用`Hexo`渲染预览就好了，效果更好。但是如果为了用在其他地方的话，这个插件还是不错的，而且它也支持其他格式的文件。

## 小结
![](/uploads/ST3.png)
最后上一张王道，这样配置一番后，确实比之前的ST要好用很多(然而代码写得还是那么烂)。




