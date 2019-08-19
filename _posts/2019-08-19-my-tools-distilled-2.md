---
layout: post
title: 我的工具记录（2）
description: 记录我使用的一些工具（2）
categories: [tools]
tags: [development, productive]
redirect_from:
  - /2019/08/19/
---

# 几个电子邮件工具的对比

[G2Rail](https://www.G2Rail.com) 启动阶段比较多关注了B2B业务。因此发送邮件是必不可少的手段。过去时间我们的Email营销工具经历了几个阶段。

## Mailchimp

早在做意启部落的时候就在用这个了，老牌工具。功能非常直观，而且发出的邮件被block的几率比价少，但是免费版的list有人数限制，只能有2000人，这对于做B2B来说，维护一个数千上万乃至更多的list是一个挑战。从创业公司的财务状况来说，按月付费来维护list也是一笔不小的成本，能省则省。

## Mailgun

Mailgun最早是我们在用户预定成功之后发电子票的服务，它每个月有10000的免费邮件额度，而且超过的部分按需付费，还是比较划算的。所以在考虑自己搭建Email Marketing服务的时候，就开始考虑是否能够考虑基于mailgun了。

## Mautic 

[Mautic](https://www.mautic.org/)是PHP写的一个开源的 ```marketing automation software```。它有一套看起来非常炫酷的根据用户与邮件的交互行为来对用户进行分类。类似于把用户置于销售漏斗中，当用户在相应的阶段，发送对应的引导邮件。不过看起来这对我们目前的业务来说有点过于复杂了。我们很多时候其实就是告诉客人，通票打折了，或者我们又上线了哪些铁路公司等等，这样似乎没有必要定义十分详细的漏斗。

此外我们需要一个类似Mailchimp的邮件所见即所得的模板设计功能，便于非技术人员使用，当时Mautic在这方便显得略有些薄弱。还有个问题是，当时还没有想到把Mautic和Mailgun结合，通过腾讯企业邮箱的SMTP服务发送，企业邮箱对这种业务有限制，所以当时设计了一个发一封信之后，过1秒再发下一封的机制，导致一封邮件要发完得一个多小时而且大量发送是失败的。算是趟了个小坑。

## Mailtrain

无意间发现[Mailtrain](https://github.com/Mailtrain-org/mailtrain)，是基于NodeJS的一个开源```newsletter application```，这个库一直在持续维护，通过Docker Compose可以不需要了解太多它的实现，简单配置之后就可以投入使用。
经过前面的两个小坑的洗礼，我意识到它可以和Mailgun结合起来。为了不影响发送电子票的sender的排名，我单独给newsletter发送服务弄了一个域名，所有的newsletter通过这个域名发送。有了Mailgun，并发发送邮件的数量得以大幅提升。另外它提供了几套基本的邮件设计template，对于目前的newsletter来说，已经非常够用了。到目前为止，这个工具一直在使用，使用效果也还不错。