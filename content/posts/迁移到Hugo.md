---
title: 迁移到Hugo
date: 2022-01-25 23:25:08
tags: [Hugo, github]
---

# 迁移blog

今天抽时间把blog从Hexo迁移到了Hugo。中间已经很多年没有更新过，想着既然想要重新开始，就干脆迁移到一个新的环境中来。

想要迁移的原因有很多：
1. 首先是真的很不习惯npm和nodejs，之前接触得很少，加上没想到Hexo即使是官方流程走下来，都会有各种报错，没能说服自己无视，又无力折腾。
2. 二来是在折腾[LeetCode Notebook in Rust](https://github.com/saukymo/leetcode-notebook-rs) 时，发现Hexo没有可以用的主题，折腾了很多要么丑，要么就是自己魔改了流程。
3. 承接上条，既然[Halfrost](https://books.halfrost.com/leetcode/)用的Hugo+hugobook，那当然最简单的思路就是跟进了。试用下来非常丝滑，于是干脆把blog一并迁移，并且更新一篇文章。

## 安装Hugo

Hugo是通过brew安装的，这一点已经观感上已经比Hexo强很多了。即使换成Python，需要我通过Pip安装的tool我也觉得很膈应。

```
brew install hugo
```

## 安装主题
```
git submodule add https://github.com/luizdepra/hugo-coder.git themes/hugo-coder
```

## 拷贝旧文档
需要用到latex的地方加上
```
math: true
```
## Github action
这一步偷懒cat其他项目的配置，多复制了一个%在文件末尾，导致github action找不到 public%目录，但是他又不报错，导致浪费将近一个小时…

```
name: github pages

on:
  push:
    branches:
      - main  # Set a branch to deploy
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          # extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

## 生成新的post

这里需要注意的是，通过这个方式生成的post会带上当地时间戳，需要在config里设置正确的时区，不然有可能生成了正确的文件，但是render完却看不到。

```
hugo new posts/xxx.md
```


## 自定义域名

在static目录下添加CNAME文件，填写自定义域名，config里的baseurl也要一并修改成这个新的自定义域名。