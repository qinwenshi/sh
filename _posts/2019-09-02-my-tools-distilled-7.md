---
layout: post
title: 我的工具记录（7）- 网络爬虫
description: 记录我使用的一些工具（7）- 网络爬虫
categories: [tools]
tags: [development, productive]
redirect_from:
  - /2019/09/02/
---

# 网络爬虫

网络爬虫可能是各行各业的热门话题。搜索引擎会通过爬虫抓取互联网数据，有些企业会抓取竞争对手的价格信息用于内部分析，有些公司会抓用户评价（比如可以在某些APP上看到大众点评的评价）等等。

## 最初德铁的爬虫

据同事说，我们在接德铁API之前，其实有一段时间网页上显示的价格是“抓取”德铁数据实现的。后来我看过它的代码是一个PHP实现的解析网页的一段脚本。这段脚本通过拼接URL，然后cURL目标网页之后解析。一段时间之后因为网页结构变化，已经不太能正常工作了。

## Go语言实现的Colly

可能有人会喜欢puppeteer或者是Google Chrome Headless版本。我个人用下来，发现[Colly](http://go-colly.org/docs/introduction/install/) 更好用，更轻量。它可以理解成上一段讲的PHP实现的爬虫的升级版。针对那种服务器端Render出来的网页十分有效。相比通过Chrome Headless启动新的tab来爬网页，Colly对爬一个新网页的overhead非常小，所以可以在配置很低的服务器上就能实现比较大的并发。

爬虫通常会遇到这些问题：
1. 因为User-Agent被服务器识别为爬虫而block
2. 需要设置请求相关的登录信息，包括Cookie，Token
3. IP地址访问次数太多而受限
4. 网页结构比较复杂，提取信息困难

问题3是另一个技术问题，在此不作讨论。Colly对1，2，4都能很好解决
Colly有个库叫 [redisstorage](github.com/gocolly/redisstorage)，通过简单设置就能够帮助Colly记住需要在浏览器保存的Cookie等信息。

```golang
  c := colly.NewCollector(colly.AllowURLRevisit(), colly.UserAgent("Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.131 Safari/537.36"))
  storage := &redisstorage.Storage{
		Address:  "127.0.0.1:6378",
		Password: "",
		DB:       0,
		Prefix:   "mysite",
	}
```

更强大的地方是，它在解析网页的时候有一系列的hook，我们只需要写jQuery类似的元素选择器，就能找到网页中的元素，提取信息，甚至填写及提交表单。

```golang
c.OnHTML("#login-form", func(form *colly.HTMLElement) { 
  fillform("shiqinwen", "123456", form)
}

e.ForEach("tbody.user-comments", func(index int, tbody *colly.HTMLElement) {

}
```

跟Colly相关或者类似的还有几个库，可以看看：

- [Surf](https://github.com/headzoo/surf)
- [Thal](https://github.com/emadehsan/thal)
- [ferret](https://github.com/MontFerret/ferret)
- goquery

## PySpider

Python领域里面也有十分出众的爬虫方案，PySpider是我做过一些评估的一个项目，很火爆。它通过配置和写一些简单的脚本就能实现网页抓取，还有界面。据说可以通过搭建一套分布式的爬虫网络，实现从不同IP地址发HTTP请求。

相比Colly而言，PySpider可能更适合于把海量的数据抓取之后，保存在文件或者数据库做进一步分析；而colly则可以适用于实时抓取，实时展示的效果。这时候就不得不提到Apify这个产品了

## Apify

这是一个很酷的创业项目，它本的理念是，通过提供十分方便的框架以及插件，来实现把任何一个网页都转化成一个可以被第三方consume的API。当然，它行走在灰色地带，毕竟爬别人的数据为己所用不是什么特别光彩的事情。。。

[Apify](apify.com) 有丰富的客户端SDK，接入也很方便，也提供云服务，把爬虫部署在他们的云里，以及IP地址切换等等事情。这个项目值得持续关注。

## Dashblock

前不久ProductHunt上推荐了一个和Apify很像的产品，名字不太好记叫 [Dashblock](https://www.producthunt.com/posts/dashblock)，它也是把网页通过便捷的技术手段，简单点击之后就可以快速部署上线工作。

## 写在最后

爬虫是一个很大的话题，除了爬虫本身之外，还有可能涉及到数据清洗、数据缓存、分析等等话题。当然，这里我们爬的网页大部分是服务器端渲染的网页。现在越来越多的网站开始转向Vue.js、React或者Angular，他们更多逻辑在客户端实现，因此直接抓可能会抓下来index.html的空壳。这时候更多的是要去inspect客户端与服务器之间的通信方式。更繁琐的是涉及到登录、授权相关的问题。也有越来越多的网站开始提供手机App，所以对手机App进行hack，找到它和服务器之间通信的部分也可以获取到相应的api——关于这个问题可以单独再写一篇。


