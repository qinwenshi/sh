---
layout: post
title: 一次iOS 13应用名称全球化翻译的调试小记
description: 一场iOS 13应用名称全球化翻译的调试小记
categories: [development]
tags: [development, flutter]
redirect_from:
  - /2019/11/09/
---

# 背景

我们发布[Xmove智行](https://apps.apple.com/us/app/xmoveapp/id1478629552?ls=1) 已经有好几个月了。最近iOS 13系统之后，有用户常常打电话进来问我们：“你们说自己叫Xmove，为啥app下载下来之后叫Runner呢？”。实际上，我们是有对APP做根据系统语言来显示不同名字的逻辑可以参考[这里](https://stackoverflow.com/questions/25736700/how-to-localise-a-string-inside-the-ios-info-plist-file)。

## 第一个坑

经过一番研究，我发现了一个很微妙的问题：国际化翻译的文件InfoPlist.strings里面的Key是[不需要空格的](https://stackoverflow.com/a/52699715)。

![BunleNameChangeSet](/image/2019-11-09/bundle-name.jpg){:width="450px"}

在把所有的strings文件修正过后，在我们的测试手机上显示已经正常了。

## 第二个坑

针对对上述Fix我们push了一个版本，以为就OK了。

结果过了两天还是有人问为啥APP下载下来之后叫Runner。于是[丹总](http://www.danielteng.com/awakener/)又在升级了版本的iOS 13上试了一下，发现果然还是没有Fix。于是我们把Google的结果翻了个底朝天，也没有啥结果。经过排除法，发现这个问题好像就是iOS 13手机真机会出现，在模拟器和低版本则没有问题。

在调试的过程中，我们逐步把CFBundleDisplayName设值的地方删除，最后发现，在根Info.plist里面的这个设置，在iOS 13中的表现不太一样，删掉两行就OK了。

![info-plist-fix](/image/2019-11-09/info-plist.jpg){:width="450px"}

后来我们推测，在较早版本的iOS里面，InfoPlist.strings也就是语言设置的值会覆盖Info.plist中设置的值；而在iOS 13中，这个行为不一样了，Info.plist中的值会被优先展示。

## 一些感想

不同的人看到这段经历的时候会有诸多质疑，有人会说，在发布前测试的不够充分、应该更早发现这个问题并解决。这是一个见仁见智的问题。如果因为这个问题影响导致我们无法持续交付[G2Rail](https://www.g2rail.com)的其他新特性，权衡之下，我们还是决定优先发布其他Feature，比如新铁路公司接入、实时出发信息等功能。等到有capacity时再来看这个问题。丹总说 Continuous over Do It Once，这也是我们的价值观所在，持续交付价值，并且容忍不完美甚至一些Broken存在。