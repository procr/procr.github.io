---
title: Blogging like a hacker
date: 2016-01-30
tag: 随笔
---

这是许久以来终于下定决心开始写的第一篇博客，回首往事，这种懒惰拖延又不精致的生活已经困扰了我好几年，希望这个博客将会是一个好的开始。

博文主要会集中描述的方面可能会有：生活，旅行，以及计算机相关的技术。

为什么取这个名字，也许是因为当时最近搭建这个博客的时候，正在单曲循环周杰伦的《手写的从前》，发现这挺符合这个博客想要达到的目的。

另外，这篇博客会简单描述一下是如何搭建的。


## Quick Start

### Hexo

常用的几个指令
``` bash
hexo s(erver):      本地server
hexo clean:         清除 public/
hexo g(enerate):    生成静态文件
hexo d(eploy):      部署到相对应的网站（比如github pages）
hexo g -d:          生成静态文件并且部署
```
More info: [hexo](https://hexo.io/zh-cn/)

### 域名重定向

搭建在github pages上的博客，访问地址一般都类似 procr.github.io，先在hexo模板的source文件夹下，放入CNAME文件，在里面写上

```bash
procr.cn
```

这告诉github pages，当访问procr.github.io这个网址的时候，会重定向到procr.cn

另外，在云解析供应商（万网），填入如下的信息：
![万网云解析](/img/yunjiexi.jpg "万网云解析")

接下来就是漫长的等待，一般需要72h以内。

TODO：
之后会写一下这个A记录等是啥意思，其实至今不是很理解。
