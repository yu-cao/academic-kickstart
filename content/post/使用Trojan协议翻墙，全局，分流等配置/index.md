---
title: '使用Trojan协议翻墙，全局，分流等配置'
subtitle: '主要针对Mac使用Trojan进行全局化分流的配置'
summary: 之前一篇文章完成了Chrome浏览器中用Trojan协议的翻墙功能，我们有些时候不单单需要浏览器，还需要比如Terminal中brew等的翻墙来更新，所以需要进行全局化
authors:
- admin
tags: ["GFW", "Trojan", "MacOS"]
categories: [GFW]
date: "2020-02-25T00:00:00Z"
lastmod: "2020-02-25T00:00:00Z"
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder. 
image:
  placement: 1
  caption: ''
  focal_point: "Center"
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

同样的，本文将基于Mac系统
经过测试，现在来说，使用mellow程序进行全局化是比较合适与方便的
mellow下载地址为：[https://github.com/mellow-io/mellow](https://github.com/mellow-io/mellow)
前一篇文章设定的port是10800，但是这里如果port设定太高似乎有问题（不知道为啥之前的10800端口一直起不来），重新改成1080的端口，设置修改如下

![](2.png)

我们动用的是socks代理，所以这里的sock要改成我们的地址与端口，具体内容参考YouTube：

[翻墙 | trojan分流和客户端使用](https://www.youtube.com/watch?v=Gkxvh6uVWU4)

然后使用Safari进行一下测试吧~

![](featured.png)

注意，使用chrome的话可以把之前设置的Proxy SwitchyOmega插件里面的自动转换改成系统代理（也就是这个插件不再起作用，全部走mellow的配置）