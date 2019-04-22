---
layout:     post
title:      "使用MediaCodeC将图片集编码为视频"
subtitle:   "将图片集高效编码为视频"
date:       2019-04-01
author:     "JadynAi"
header-img: ""
tags:
    - Android
    - 视频
    - MediaCodeC
---

**原创文章，转载请联系作者**
>绿生莺啼春正浓，钗头青杏小，绿成丛。
>玉船风动酒鳞红。歌声咽，相见几时重？

### 提要
这是`MediaCodeC`系列的第三章，主题是如何使用MediaCodeC将图片集编码为视频文件。在Android多媒体的处理上，MediaCodeC是一套非常有用的API。此次实验中，所使用的图片集正是[MediaCodeC硬解码视频，并将视频帧存储为图片文件](https://ailoli.me/2019/01/25/MediaCodeC-Decode-1/)文章中，对视频解码出来的图片文件集，总共332张图片帧。<br>若是对`MediaCodeC`视频解码感兴趣的话，也可以浏览之前的文章：[MediaCodeC解码视频指定帧，迅捷、精确](https://ailoli.me/2019/02/09/MediaCodeC-frame/)

### 核心流程
`MediaCodeC`的常规工作流程是：拿到可用输入队列，填充数据；拿到可用输出队列，取出数据。一般情况下，填充和取出两个动作并不是同步的，也就是说并不是压入一帧数据，就能拿出一帧数据。当然，除了编码的视频每一帧都是关键帧的情况下。<br>
Android官方文档给出的一种同步代码写法，这也是比较常用到的代码：

```
for (;;) {
	//拿到可用InputBuffer的id
  int inputBufferId = codec.dequeueInputBuffer(timeoutUs);
  if (inputBufferId >= 0) {
    ByteBuffer inputBuffer = codec.getInputBuffer(…);
    // inputBuffer 填充数据
    codec.queueInputBuffer(inputBufferId, …);
  }
  // 查询是否有可用的OutputBuffer
  int outputBufferId = codec.dequeueOutputBuffer(…);
```
而本篇文章将会使用另一种方式进行编码，即使用Surface代替InputBuffer来实现数据的填充。此次编码使用`createInputSurface `作为硬编码Input Surface。<br>这里我画了一张简单的工作流程图：![](https://raw.githubusercontent.com/JadynAi/MediaLearn/master/pic/mediacodec_encoder.png)
整体流程上其实和普通的MediaCodeC工作流程差不多，只不过是将输入源由Buffer换成了Surface。那么既然使用了Surface作为输入源，该如何向Surface填充数据呢？<br>答案就是使用OpenGL。

### 流程详解
在详解流程前，有一点要注意的是，工作流程中所有环节都必须处在同一线程。



### 结语
[此处有项目地址，点击传送](https://github.com/JadynAi/MediaLearn/blob/master/mediakit/src/main/java/com/jadyn/mediakit/video/decode/VideoDecoder2.kt)


