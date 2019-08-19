---
layout: post
title: 我的工具记录（3）
description: 记录我使用的一些工具（3）
categories: [tools]
tags: [development, productive]
redirect_from:
  - /2019/08/20/
---

# 搭建Git服务器

开源产品会用GitHub或者Bitbucket公开代码仓库。最近GitHub开放私有仓库免费，似乎有些自建Git服务显得没有必要。我发现github有时候在境内访问比较慢，另外越来越多github上涉及到了政治敏感的话题，说不定哪天它就被block了。Bitbucket也是类似的情况，有时候慢的不能忍受。所以出于种种考虑，还是自建git吧。

## GitLab

虽然自己很多时间是在写Ruby代码，还是不得不吐槽，GitLab真是太占资源了，就一台服务器给它，有时候还哼哧哼哧的卡住。所以一开始用了一段时间的自建GitLab之后就考虑要放弃了。

## GoGS

我对Go语音产生好感最主要是源于[GoGS](https://gogs.io/)。一开始看到这个名字还没理解是个啥，转念一想才明白是Go Git Service。同样道理，GOGS也可以直接通过Docker启动。通过Docker启动之后还要折腾一下把host的22端口转发给docker启动后的端口。一般ssh是22，但是docker里面的ssh端口一般不会直接绑定到22，所以要做一个端口共享，设置过程有点小折腾。如果需要帮助可以给我发信。

## Gitea

[Gitea](https://gitea.io/en-us/)其实是基于GOGS的，最早的时候GOGS出来，一帮人觉得很酷，于是纷纷想给它加更炫酷的UI和相关功能，结果GOGS好像对这些改动不是十分欢迎，于是他们就单独搞了Gitea这个项目。现在GOGS在Github上有3万多Star，Gitea也有一万五千Star。

## 写在后面
相比而言，我个人更喜欢GOGS，十分简单，不花哨，运行到现在近3年了，基本上没出过啥问题，十分稳定，几乎都感觉不到它的存在。这也符合我对软件和工具的理解——软件和工具，应该是像人的手或者脚，甚至是水和空气。大多数时候，人们甚至都不一定意识到它的存在，它静静的帮人进入一种flow的状态，顺理成章的达成自己的目的。