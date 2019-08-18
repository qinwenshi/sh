---
layout: post
title: 我的工具记录（1）
description: 记录我使用的一些工具（1）
categories: [tools]
tags: [development, productive]
redirect_from:
  - /2019/09/19/
---

不知不觉已经有近800个Star的代码仓库自建或者fork的仓库也有近400个。打算抽空梳理一下这些，顺便记下来一些用法，以及为啥当初会关注这个库。

## 剪切板工具
我一直在用 Mac上的 [ClipMenu](https://github.com/naotaka/ClipMenu)，非常简介直观。在Mac上通过 ```cmd+shift+v```就可以调出所有过去曾经复制的内容，包括文字、图片等等。

![ClipMenu](/image/2019-08-19/clipmenu.jpg){:width="450px"}

前两天给同事分享屏幕，他问起Windows系统上是否有类似的工具，正好看到
在Windows上可以用 [CopyQ](https://github.com/hluk/CopyQ/)。它其实也是支持各种系统的，包括Mac。在虚拟机里面试了一下效果也还不错。
可以快速设置全剧快捷键，比如“在鼠标处显示主窗口“，有更多快捷键可以自己设置。比如“粘贴并复制下一个”。

![options](/image/2019-08-19/copy-q-options.png){:width="450px"}

## Git客户端

命令行里面一般用git一系列的组合命令来做代码提交，查看历史等等。文件多的时候很容易出错。SourceTree是挺不错的GUI工具，中规中矩。不过想要炫酷一些的话，SourceTree就不够了。

[LazyGit](https://github.com/jesseduffield/lazygit) 是用Go写的一个很轻量级的工具基于gocui的命令行工具。这一年它也不断有更新。启动之后会在命令行启动一个GUI，可以查看历史、git pull、git push等

![Gif](https://github.com/jesseduffield/lazygit/raw/master/docs/resources/lazygit-example.gif)

前两天在Facebook上看到一个界面也很炫的客户端 [gitkraken](https://www.gitkraken.com/)，看上去不错。它集成了一个叫Glo的东东，看起来像是集成的Trello。先玩一段时间看看。

![Gitkraken](/image/2019-08-19/gitkraken.png){:width="450px"}
