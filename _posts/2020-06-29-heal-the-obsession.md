---
layout: post
title: 拯救强迫症
description: 拯救强迫症
categories: [thinking]
tags: [thinking, productive]
redirect_from:
  - /2020/06/29/
---

看看[Xmove智行](https://app.g2rail.com)的这个两个截图，能看到有啥变化不？

图一：修改前

![修改前]](/image/2020-06-29/1.jpg){:width="450px"}

图二：修改后

![修改后]](/image/2020-06-29/2.jpg){:width="450px"}

其实这里面有一个很小的overlapping，在红圈标出来的小圆点，被右边的时间盖住了大约3个像素。当然，如果是屏幕相对更窄一些的手机会看得更加明显。

图三：更加明显

![明显的版本]](/image/2020-06-29/3.jpg){:width="450px"}

![开发版]](/image/2020-06-29/4.jpg){:width="450px"}。

在开发版本上看得更加明显。上个月跟Daniel有过一次讨论，其实这里有个稍微有点棘手的前端技术问题——出发时间、到达时间、价格三个元素分别作为三列在卡片中展示，中间换乘的小widget暂时放在第一个column展示。昨天散步的时候忽然间想到，可以通过Flutter的Stack控件，设置overflow为visible，加上Positioned将内部控件定位与靠右就可以实现了。

![挑战]](/image/2020-06-29/6.png){:width="450px"}

当时看到这个问题的时候，正在集中心力完善[Serendipity APP](https://play.google.com/store/apps/details?id=com.g2rail.serendipity)，所以暂时搁置了。

这也是我们在打穿的时候的一个实践，把暂时没有结论的问题，放在脑后，睡觉的时候、散步的时候、或者打球的时候留着一根弦，经过一段时间的孵化，常常会有满意的结果。

最后修改完就好了，“强迫症患者”可能会比较喜欢。

![最终版]](/image/2020-06-29/5.jpg){:width="450px"}

