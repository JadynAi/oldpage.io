---
layout:     post
title:      "提供一个Glide灵活加载圆角图片的方法"
subtitle:   "四个圆角灵活切换"
date:       2018-07-01
author:     "JadynAi"
header-img: "img/post-bg-2015.jpg"
tags:
    - 技术讨论
    - Glide
---
**原创文章，转载请联系作者**
### 前言
`Glide`是目前使用的颇为广泛的图片加载框架，同时也是Google官方推荐使用的。在图片处理方面，它提供了很多不错的功能。
### 如何才能灵活
圆角图片显示，大概是很多APP都会出现的UI设计了，`Glide`本身也提供了圆角图片的加载方式——*但也只是简单的四圆角*。实际项目开发中，并不能应付多变的产品需求和善变的UI设计师了。譬如有时候，只需要顶部展示圆角，有时候又只需要左侧展示圆角等等。<br>那么，本次的方法适用于使用了`Glide`作为项目图片框架的小可爱们，开发者可以对四个圆角进行单独设置，不仅仅是显示隐藏，每个圆角的半径亦是独立存在的。
### 使用文档
`Glide`有对外暴露一个方法，可以在图片显示前，对图片作转换处理——就是`Transformations`。有关此方面的文字，小可爱们可以看看这篇——[Glide - 自定义转换](https://mrfu.me/2016/02/28/Glide_Custom_Transformations/)。本文的`RoundCorner`就是继承了`BitmapTransformation`类来实现的。它对外提供两个构造函数，一个构造函数有四个参数，分别是`leftTop:左上角`、`rightTop:右上角`、`leftBottom:左下角`、`rightBottom：右下角`。可以供外部，灵活的去选择设置哪几个圆角需要去展示，四个圆角的半径大小。另一个函数，只提供一个参数就是同时设置四个圆角，当然这是用于四个圆角同时展示且半径相同的情况下。于此同时，构造函数中只需要传数值即可，类内部已经做了dp处理。<br>下面做一下简单的展示。

- 加载普通圆角图片

```
Glide.with(this).load("http://p15.qhimg.com/bdm/720_444_0/t01b12dfd7f42342197.jpg")
        .apply(RequestOptions.bitmapTransform(RoundCorner(20f)))
        .into(img)
```
![](https://wx3.sinaimg.cn/mw690/a28b91d8gy1fsuqu1pfaoj20qm0h0goy.jpg)

- 只是顶部圆角

```
Glide.with(this).load("http://p15.qhimg.com/bdm/720_444_0/t01b12dfd7f42342197.jpg")
        .apply(RequestOptions.bitmapTransform(RoundCorner(leftTop = 20f, rightTop = 20f)))
        .into(img)
```
![](https://mmbiz.qpic.cn/mmbiz_png/jqw9LvhdsxJMVGR4oYRkW5Gk7G3EBkdv4IHqEN8ydtRhEw64Tofh36Wgl7ricaKYQ7n74VF1uia8IWwDGO3icx3IA/0?wx_fmt=png)

- 只是左侧圆角

```
Glide.with(this).load("http://p15.qhimg.com/bdm/720_444_0/t01b12dfd7f42342197.jpg")
        .apply(RequestOptions.bitmapTransform(RoundCorner(leftTop = 20f, leftBottom = 20f)))
        .into(img)
```
![](https://mmbiz.qpic.cn/mmbiz_png/jqw9LvhdsxJMVGR4oYRkW5Gk7G3EBkdvoPj9Uz8kvjGadIhNdLmPCI7BunGwxOHt6UfibIMN6vIlz4Zk16gyakQ/0?wx_fmt=png)

以上就是一次简单的展示了，如果你想更加灵活的加载圆角图片,选择这个方法没有错。代码在这里，传送门——[RoundCorner](https://github.com/JadynAi/KotlinDiary/blob/master/app/src/main/java/com/jadynai/kotlindiary/function/img/RoundCorner.kt)

### 结语
以上<br>![](http://JadynAi.github.io/img/wechat_official.png)
