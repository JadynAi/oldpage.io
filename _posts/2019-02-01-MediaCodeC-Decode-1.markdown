---
layout:     post
title:      "MediaCodeC硬解码视频，并将视频帧存储为图片文件"
subtitle:   "使用两种不同的方法"
date:       2019-02-01
author:     "JadynAi"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - 视频
---

**原创文章，转载请联系作者**
>醉拍春衫惜旧香，天将离恨恼疏狂。
>年年陌上生秋草，日日楼中到夕阳。

### 目的
- MediaCodeC搭配MediaExtractor将视频完整解码
- 视频帧存储为JPEG文件
- 使用两种方式达成
    - 硬编码输出数据二次封装为YuvImage，并直接输出为JPEG格式文件
    - 硬编码搭配Surface，用OpenGL封装为RGBA数据格式，再利用Bitmap压缩为图片文件
    - 二者皆可以调整图片输出质量
      
### 参考

- YUV的处理方式，强推大家观看这篇文章[高效率得到YUV格式帧](https://www.polarxiong.com/archives/Android-MediaCodec%E8%A7%86%E9%A2%91%E6%96%87%E4%BB%B6%E7%A1%AC%E4%BB%B6%E8%A7%A3%E7%A0%81-%E9%AB%98%E6%95%88%E7%8E%87%E5%BE%97%E5%88%B0YUV%E6%A0%BC%E5%BC%8F%E5%B8%A7-%E5%BF%AB%E9%80%9F%E4%BF%9D%E5%AD%98JPEG%E5%9B%BE%E7%89%87-%E4%B8%8D%E4%BD%BF%E7%94%A8OpenGL.html)，绝对整的明明白白
- OpenGL的处理方式，当然是最出名的[BigFlake](https://bigflake.com/mediacodec/ExtractMpegFramesTest_egl14.java.txt)，硬编码相关的示例代码很是详细

### 解码效率分析
- 参考对象为一段约为13.8s，H.264编码，FPS为24，72*1280的MPEG-4的视频文件。[鸭鸭戏水视频](https://raw.githubusercontent.com/JadynAi/MediaLearn/master/pic/yazi.mp4)
	- 此视频的视频帧数为332	
- 略好点的设备解码时间稍短一点。但两种解码方式的效率对比下来，`OpenGl渲染`耗费的时间比`YUV转JPEG`多。
	- 另：差一点的设备上，这个差值会被提高，约为一倍多。较好的设备，则小于一倍。
	
### 实现过程
对整个视频的解析，以及压入MediaCodeC输入队列都是通用步骤。

```

mediaExtractor.setDataSource(dataSource)
// 查看是否含有视频轨
val trackIndex = mediaExtractor.selectVideoTrack()
if (trackIndex < 0) {
    throw RuntimeException("this data source not video")
}
mediaExtractor.selectTrack(trackIndex)
      
       
fun MediaExtractor.selectVideoTrack(): Int {
    val numTracks = trackCount
    for (i in 0 until numTracks) {
        val format = getTrackFormat(i)
        val mime = format.getString(MediaFormat.KEY_MIME)
        if (mime.startsWith("video/")) {
            return i
        }
    }
    return -1
}

```

配置MediaCodeC解码器，将解码输出格式设置为**COLOR_FormatYUV420Flexible**，这种模式几乎所有设备都会支持。<br>**使用OpenGL渲染的话，MediaCodeC要配置一个输出Surface。使用YUV方式的话，则不需要配置**

```
		outputSurface = if (isSurface) OutputSurface(mediaFormat.width, mediaFormat.height) else null

        // 指定帧格式COLOR_FormatYUV420Flexible,几乎所有的解码器都支持
        if (decoder.codecInfo.getCapabilitiesForType(mediaFormat.mime).isSupportColorFormat(defDecoderColorFormat)) {
            mediaFormat.setInteger(MediaFormat.KEY_COLOR_FORMAT, defDecoderColorFormat)
            decoder.configure(mediaFormat, outputSurface?.surface, null, 0)
        } else {
            throw RuntimeException("this mobile not support YUV 420 Color Format")
        }

        val startTime = System.currentTimeMillis()
        Log.d(TAG, "start decode frames")
        isStart = true
        val bufferInfo = MediaCodec.BufferInfo()
        // 是否输入完毕
        var inputEnd = false
        // 是否输出完毕
        var outputEnd = false
        decoder.start()
        var outputFrameCount = 0

        while (!outputEnd && isStart) {
            if (!inputEnd) {
                val inputBufferId = decoder.dequeueInputBuffer(DEF_TIME_OUT)
                if (inputBufferId >= 0) {
                    // 获得一个可写的输入缓存对象
                    val inputBuffer = decoder.getInputBuffer(inputBufferId)
                    // 使用MediaExtractor读取数据
                    val sampleSize = videoAnalyze.mediaExtractor.readSampleData(inputBuffer, 0)
                    if (sampleSize < 0) {
                        // 2019/2/8-19:15 没有数据
                        decoder.queueInputBuffer(inputBufferId, 0, 0, 0L,
                                MediaCodec.BUFFER_FLAG_END_OF_STREAM)
                        inputEnd = true
                    } else {
                        // 将数据压入到输入队列
                        val presentationTimeUs = videoAnalyze.mediaExtractor.sampleTime
                        decoder.queueInputBuffer(inputBufferId, 0,
                                sampleSize, presentationTimeUs, 0)
                        videoAnalyze.mediaExtractor.advance()
                    }
                }
            }
```

可以大致画一个流程图如下：<br>

![](https://raw.githubusercontent.com/JadynAi/MediaLearn/master/pic/decode_line.png)

#### YUV
通过以上通用的步骤后，接下来就是对MediaCodeC的输出数据作YUV处理了。步骤如下：

1.使用MediaCodeC的`getOutputImage (int index)`函数，得到一个只读的Image对象，其包含原始视频帧信息。
> By：当MediaCodeC配置了输出Surface时，此值返回null

2.将Image得到的数据封装到YuvImage中，再使用YuvImage的`compressToJpeg `方法压缩为JPEG文件
> YuvImage的封装，官方文档有这样一段描述：`Currently only ImageFormat.NV21 and ImageFormat.YUY2 are supported`。
>YuvImage只支持*NV21*或者*YUY2*格式，所以还可能需要对Image的原始数据作进一步处理，将其转换为*NV21*的Byte数组 

##### 读取Image信息并封装为Byte数组
此次演示的机型，反馈的Image格式如下：<br>
> getFormat = 35<br>getCropRect().width()=720<br>getCropRect().height()=1280

35代表`ImageFormat.YUV_420_888格式`。Image的`getPlanes`会返回一个数组，其中0代表Y，1代表U，2代表V。由于是420格式，那么四个Y值共享一对UV分量，比例为4：1。<br>**代码如下，参考[YUV_420_888编码Image转换为I420和NV21格式byte数组](https://www.polarxiong.com/archives/Android-YUV_420_888%E7%BC%96%E7%A0%81Image%E8%BD%AC%E6%8D%A2%E4%B8%BAI420%E5%92%8CNV21%E6%A0%BC%E5%BC%8Fbyte%E6%95%B0%E7%BB%84.html),不过我这里只保留了NV21格式的转换**

```
fun Image.getDataByte(): ByteArray {
    val format = format
    if (!isSupportFormat()) {
        throw RuntimeException("image can not support format is $format")
    }
    // 指定了图片的有效区域，只有这个Rect内的像素才是有效的
    val rect = cropRect
    val width = rect.width()
    val height = rect.height()
    val planes = planes
    val data = ByteArray(width * height * ImageFormat.getBitsPerPixel(format) / 8)
    val rowData = ByteArray(planes[0].rowStride)

    var channelOffset = 0
    var outputStride = 1
    for (i in 0 until planes.size) {
        when (i) {
            0 -> {
                channelOffset = 0
                outputStride = 1
            }
            1 -> {
                channelOffset = width * height + 1
                outputStride = 2
            }
            2 -> {
                channelOffset = width * height
                outputStride = 2
            }
        }

        // 此时得到的ByteBuffer的position指向末端
        val buffer = planes[i].buffer
        //  行跨距
        val rowStride = planes[i].rowStride
        // 行内颜色值间隔，真实间隔值为此值减一
        val pixelStride = planes[i].pixelStride

        val TAG = "getDataByte"

        Log.d(TAG, "planes index is  $i")
        Log.d(TAG, "pixelStride $pixelStride")
        Log.d(TAG, "rowStride $rowStride")
        Log.d(TAG, "width $width")
        Log.d(TAG, "height $height")
        Log.d(TAG, "buffer size " + buffer.remaining())

        val shift = if (i == 0) 0 else 1
        val w = width.shr(shift)
        val h = height.shr(shift)
        buffer.position(rowStride * (rect.top.shr(shift)) + pixelStride +
                (rect.left.shr(shift)))
        for (row in 0 until h) {
            var length: Int
            if (pixelStride == 1 && outputStride == 1) {
                length = w
                // 2019/2/11-23:05 buffer有时候遗留的长度，小于length就会报错
                buffer.getNoException(data, channelOffset, length)
                channelOffset += length
            } else {
                length = (w - 1) * pixelStride + 1
                buffer.getNoException(rowData, 0, length)
                for (col in 0 until w) {
                    data[channelOffset] = rowData[col * pixelStride]
                    channelOffset += outputStride
                }
            }

            if (row < h - 1) {
                buffer.position(buffer.position() + rowStride - length)
            }
        }
    }
    return data
}
```
##### 最后封装YuvImage并压缩为文件

```
	val rect = image.cropRect
    val yuvImage = YuvImage(image.getDataByte(), ImageFormat.NV21, rect.width(), rect.height(), null)
    yuvImage.compressToJpeg(rect, 100, fileOutputStream)
    fileOutputStream.close()
```

#### MediaCodeC配置输出Surface，使用OpenGL渲染
OpenGL的环境搭建和渲染代码不再赘述，只是强调几个点：
- 渲染纹理的线程一定要和MediaCodeC配置Surface的线程保持一致
- 在渲染纹理代码前，一定要调用MediaCodeC的`releaseOutputBuffer`函数，将输出数据及时渲染到输出Surface上，否则Surface内的纹理将不会收到任何数据

##### 获得可用的RGBA数据，使用Bitmap压缩为指定格式文件

```
fun saveFrame(fileName: String) {
        pixelBuf.rewind()
        GLES20.glReadPixels(0, 0, width, height, GLES20.GL_RGBA, GLES20.GL_UNSIGNED_BYTE, pixelBuf)
        var bos: BufferedOutputStream? = null
        try {
            bos = BufferedOutputStream(FileOutputStream(fileName))
            val bmp = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888)
            pixelBuf.rewind()
            bmp.copyPixelsFromBuffer(pixelBuf)
            bmp.compress(Bitmap.CompressFormat.JPEG, 100, bos)
            bmp.recycle()
        } finally {
            bos?.close()
        }
    }
```

### 结果分析
到目前为止，针对样例视频，`YUV`解码出来的视频帧亮度会稍低一点，且图片边缘处有细微的失真。`OpenGL渲染`解码的视频帧会明亮一些，放大三四倍边缘无失真。后续会继续追踪这个问题，会使用`FFmpeg`解码来作为对比。

### 结语
[此处有项目地址，点击传送](https://github.com/JadynAi/MediaLearn/tree/master/app/src/main/java/com/jadyn/ai/medialearn/decode)

<br>**以上**<br>**原创不易，大家走过路过看的开心，可以适当给个一毛两毛聊表心意**<br>![](http://JadynAi.github.io/img/person_wechat.jpg)


