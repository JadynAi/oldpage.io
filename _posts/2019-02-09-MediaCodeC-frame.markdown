---
layout:     post
title:      "MediaCodeC解码视频指定帧，迅捷、精确"
subtitle:   "使用MediaCodeC提高音视频开发效率"
date:       2019-02-09
author:     "JadynAi"
header-img: ""
tags:
    - Android
    - 视频
---

**原创文章，转载请联系作者**
>若待明朝风雨过，人在天涯！春在天涯

### 提要
最近在整理硬编码**MediaCodec**相关的学习笔记，以及代码文档，分享出来以供参考。本人水平有限，项目难免有思虑不当之处，若有问题可以提`Issues`。[项目地址传送门](https://github.com/JadynAi/MediaLearn/blob/master/mediakit/src/main/java/com/jadyn/mediakit/video/decode/VideoDecoder2.kt)<br>此篇文章，主要是分享如何用`MediaCodeC`解码视频指定时间的一帧，回调Bitmap对象。之前还有一篇[MediaCodeC硬解码视频，并将视频帧存储为图片文件](http://ailoli.me/2019/02/01/MediaCodeC-Decode-1/)，主要内容是将视频完整解码，并存储为JPEG文件，大家感兴趣可以去看一看。

### 如何使用
`VideoDecoder2`上手简单直接，首先需要创建一个解码器对象：

```
val videoDecoder2 = VideoDecoder2(dataSource)
```
>dataSoure就是视频文件地址

解码器会在对象创建的时候，对视频文件进行分析，得出时长、帧率等信息。有了解码器对象后，在需要解码帧的地方，直接调用函数：

```
videoDecoder2.getFrame(time, { it->
					//成功回调，it为对应帧Bitmap对象
                }, {
                 //失败回调
              })
                
```
> time 接受一个Float数值，级别为秒

`getFrame`函数式一个异步回调，会自动回调到主线程里来。同时这个函数也没有**过度调用限制**。也就是说——，你可以频繁调用而不用担心出现其他问题。

### 代码结构、实现过程
#### 代码结构
`VideoDecoder2 `目前只支持硬编码解码，在某些机型或者版本下，可能会出现兼容问题。后续会继续补上软解码的功能模块。<br>先来看一下`VideoDecoder2 `的代码框架，有哪些类构成，以及这些类起到的作用。
![](https://raw.githubusercontent.com/JadynAi/MediaLearn/master/pic/code_framework.png)
在`VideoDecoder2 `中，`DecodeFrame`承担着核心任务，由它发起这一帧的解码工作。获取了目标帧的YUV数据后；由`GLCore`来将这一帧转为Bitmap对象，它内部封装了**OpenGL**环境的搭建，以及配置了`Surface`供给`MediaCodeC`使用。<br>`FrameCache`主要是做着缓存的工作，内部有内存缓存*LruCache*以及磁盘缓存*DiskLruCache*，因为缓存的存在，很大程度上提高了二次读取的效率。
#### 工作流程
`VideoDecoder2`的工作流程，是一个线性任务队列串行的方式。其工作流程图如下：
![](https://raw.githubusercontent.com/JadynAi/MediaLearn/master/pic/decode_frame_work.png)
具体流程：<br>

- 1.当执行`getFrame`函数时，首先从缓存从获取这一帧的图片缓存。
- 2.如果缓存中没有这一帧的缓存，那么首先判断任务队列中**正在执行的任务**是否和此时需要的任务重复，如果不重复，则创建一个`DecodeFrame`任务加入队列。
- 3.任务队列的任务是在一个特定的子线程内，线性执行。新的任务会被加入队列尾端，而已有任务则会被提高优先级，移到队列中`index为1`的位置。
- 4、`DecodeFrame `获取到这一帧的Bitmap后，会将这一帧缓存为内存缓存，并在会在**缓存线程**内作磁盘缓存，方便二次读取。

接下来分析一下，实现过程中的几个重要的点。

#### 实现过程
- 如何定位和目标时间戳相近的采样点
- 如何使用`MediaCodeC`获取视频特定时间帧
- 缓存是如何工作，起到的作用有哪些

##### 定位精确帧
**精确**其实是一个相对而言的概念，`MediaExtractor`的`seekTo`函数，有三个可供选择的标记：SEEK_TO_PREVIOUS_SYNC, SEEK_TO_CLOSEST_SYNC, SEEK_TO_NEXT_SYNC，分别是seek指定帧的上一帧，最近帧和下一帧。<br>其实，`seekTo`并无法每次都准确的跳到指定帧，这个函数只会seek到目标时间的最接近的（CLOSEST）、上一帧（PREVIOUS）和下一帧（NEXT）。因为视频编码的关系，解码器只会从关键帧开始解码，也就是I帧。因为只有I帧才包含完整的信息。而P帧和B帧包含的信息并不完全，只有依靠前后帧的信息才能解码。所以这里的解决办法是：先定位到目标时间的上一帧，然后`advance`，直到读取的时间和目标时间的差值最小，或者读取的时间和目标时间的差值小于**帧间隔**。

```
val MediaFormat.fps: Int
    get() = try {
        getInteger(MediaFormat.KEY_FRAME_RATE)
    } catch (e: Exception) {
        0
    }

/*
    * 
    * return : 每一帧持续时间，微秒
    * */
    val perFrameTime by lazy {
        1000000L / mediaFormat.fps
    }

/*
    * 
    * 查找这个时间点对应的最接近的一帧。
    * 这一帧的时间点如果和目标时间相差不到 一帧间隔 就算相近
    * 
    * maxRange:查找范围
    * */
    fun getValidSampleTime(time: Long, @IntRange(from = 2) maxRange: Int = 5): Long {
        checkExtractor.seekTo(time, MediaExtractor.SEEK_TO_PREVIOUS_SYNC)
        var count = 0
        var sampleTime = checkExtractor.sampleTime
        while (count < maxRange) {
            checkExtractor.advance()
            val s = checkExtractor.sampleTime
            if (s != -1L) {
                count++
                // 选取和目标时间差值最小的那个
                sampleTime = time.minDifferenceValue(sampleTime, s)
                if (Math.abs(sampleTime - time) <= perFrameTime) {
                    //如果这个差值在 一帧间隔 内，即为成功
                    return sampleTime
                }
            } else {
                count = maxRange
            }
        }
        return sampleTime
    }
```
> 帧间隔其实就是：1s/帧率

##### 使用`MediaCodeC`解码指定帧
获取到相对精确的采样点（帧）后，接下来就是使用`MediaCodeC`解码了。首先，使用`MediaExtractor`的`seekTo`函数定位到目标采样点。

```
mediaExtractor.seekTo(time, MediaExtractor.SEEK_TO_PREVIOUS_SYNC)
```
然后`MediaCodeC`将`MediaExtractor`读取的数据压入输入队列，不断循环，直到拿到想要的目标帧的数据。

```
/*
* 持续压入数据，直到拿到目标帧
* */
private fun handleFrame(time: Long, info: MediaCodec.BufferInfo, emitter: ObservableEmitter<Bitmap>? = null) {
    var outputDone = false
    var inputDone = false
    videoAnalyze.mediaExtractor.seekTo(time, MediaExtractor.SEEK_TO_PREVIOUS_SYNC)
    while (!outputDone) {
        if (!inputDone) {
            decoder.dequeueValidInputBuffer(DEF_TIME_OUT) { inputBufferId, inputBuffer ->
                val sampleSize = videoAnalyze.mediaExtractor.readSampleData(inputBuffer, 0)
                if (sampleSize < 0) {
                    decoder.queueInputBuffer(inputBufferId, 0, 0, 0L,
                            MediaCodec.BUFFER_FLAG_END_OF_STREAM)
                    inputDone = true
                } else {
                    // 将数据压入到输入队列
                    val presentationTimeUs = videoAnalyze.mediaExtractor.sampleTime
                    Log.d(TAG, "${if (emitter != null) "main time" else "fuck time"} dequeue time is $presentationTimeUs ")
                    decoder.queueInputBuffer(inputBufferId, 0,
                            sampleSize, presentationTimeUs, 0)
                    videoAnalyze.mediaExtractor.advance()
                }
            }
  
        decoder.disposeOutput(info, DEF_TIME_OUT, {
            outputDone = true
        }, { id ->
            Log.d(TAG, "out time ${info.presentationTimeUs} ")
            if (decodeCore.updateTexture(info, id, decoder)) {
                if (info.presentationTimeUs == time) {
                    // 遇到目标时间帧，才生产Bitmap
                    outputDone = true
                    val bitmap = decodeCore.generateFrame()
                    frameCache.cacheFrame(time, bitmap)
                    emitter?.onNext(bitmap)
                }
            }
        })
    }
    decoder.flush()
}
```
>需要注意的是，解码的时候，并不是压入一帧数据，就能得到一帧输出数据的。<br>常规的做法是，持续不断向输入队列填充帧数据，直到拿到想要的目标帧数据。<br>原因还是因为视频帧的编码，并不是每一帧都是关键帧，有些帧的解码必须依靠前后帧的信息。

##### 缓存
- LruCache，内存缓存
- DiskLruCache

LruCache自不用多说，磁盘缓存使用的是著名的[DiskLruCache](https://github.com/JakeWharton/DiskLruCache/tree/master/src/main/java/com/jakewharton/disklrucache)。缓存在`VideoDecoder2`中占有很重要的位置，它有效的提高了解码器二次读取的效率，从而不用多次解码以及使用`OpenGL`绘制。<br>
>之前在`Oppo R15`的测试机型上，进行了一轮解码测试。<br>使用`MediaCodeC`解码一帧到到的Bitmap，大概需要**100~200ms**的时间。<br>而使用磁盘缓存的话，读取时间大概在**50~60ms**徘徊，效率增加了一倍。

在磁盘缓存使用的过程中，有对[DiskLruCache](https://github.com/JakeWharton/DiskLruCache/tree/master/src/main/java/com/jakewharton/disklrucache)进行二次封装，内部使用单线程队列形式。进行磁盘缓存，对外提供了异步和同步两种方式获取缓存。可以直接搭配`DiskLruCache`使用——[DiskCacheAssist.kt](https://raw.githubusercontent.com/JadynAi/MediaLearn/master/mediakit/src/main/java/com/jadyn/mediakit/video/decode/DiskCacheAssist.kt)

### 总结
到目前为止，视频解码的部分已经完成。上一篇是对视频完整解码并存储为图片文件，[MediaCodeC硬解码视频，并将视频帧存储为图片文件](http://ailoli.me/2019/02/01/MediaCodeC-Decode-1/)，这一篇是解码指定帧。音视频相关的知识体系还很大，会继续学习下去。

### 结语
[此处有项目地址，点击传送](https://github.com/JadynAi/MediaLearn/blob/master/mediakit/src/main/java/com/jadyn/mediakit/video/decode/VideoDecoder2.kt)

<br>**以上**<br>**原创不易，大家走过路过看的开心，可以适当给个一毛两毛聊表心意**<br>![](http://JadynAi.github.io/img/person_wechat.jpg)


