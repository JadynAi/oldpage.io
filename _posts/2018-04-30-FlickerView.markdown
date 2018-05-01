---
layout:     post
title:      "仿ios京东启动页“跳过”效果"
subtitle:   "寒光亭下水如天，飞起沙鸥一片"
date:       2018-04-30
author:     "JadynAi"
header-img: "img/20180430-blog-header-bg.jpg"
tags:
    - Android动画
---

> 天共水，水远与天连<br>天净水平寒月漾，水光月色两相兼

**简单展示一下动画效果：**<br>![](https://wx1.sinaimg.cn/mw690/a28b91d8gy1fqums8z9rig205f05wmxe.gif)

**[项目地址在此](https://github.com/JadynAi/LoadingLovely/blob/master/app/src/main/java/com/example/jadynai/loadinglovely/flicker/TextFlickerView.java)，大家若是喜欢的话，不妨点个赞吧**<br>
*好了，简单阐述一下本次动画的原理：*
> 光的效果使用`Paint`设置`Shader`来实现，具体则是`LinearGradient`水平渐变渲染。<br>光影的平移依赖于`LinearGradient`的`setLocalMatrix`，通过`Matrix`的`translate`来促使光影移动。

#### 在自定义View的`onSizeChanged(int w, int h, int oldw, int oldh)`方法内初始化引擎：

```
@Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        super.onSizeChanged(w, h, oldw, oldh);
        tryInitEngine(w);
    }
    
     private void tryInitEngine(int w) {
        if (mShadowMatrix == null) {
            if (w > 0) {
                //控制阴影的Matrix，通过Matrix的变化来实现闪光的滑过效果
                mShadowMatrix = new Matrix();
                //因为使用了LinearGradient,所以Paint本身的color将毫无意义，所以colors的起始点的色值必须和本来色值一致
                int currentTextColor = getCurrentTextColor();
                //渐变色层.x0,y0是起点坐标，x1，y1是终点坐标
                mLinearGradient = new LinearGradient(0, 0, 50, 0, new int[] {currentTextColor, Color.GREEN, currentTextColor},
                        null, Shader.TileMode.CLAMP);
                //画笔设置Shader
                getPaint().setShader(mLinearGradient);
                //使用属性动画作为引擎，数值从-SHADOW变化到TextView本身的宽度。间隔时间未1500ms
                mValueAnimator = ValueAnimator.ofFloat(-50, w).setDuration(1500);
                mValueAnimator.setInterpolator(new LinearInterpolator());
                mValueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
                    @Override
                    public void onAnimationUpdate(ValueAnimator animation) {
                        float value = (float) animation.getAnimatedValue();
                        //Matrix移动来实现闪光滑动
                        mShadowMatrix.setTranslate(value, 0);
                        invalidate();
                    }
                });
                mValueAnimator.addListener(new AnimatorListenerAdapter() {
                    @Override
                    public void onAnimationRepeat(Animator animation) {
                        super.onAnimationRepeat(animation);
                        mShadowMatrix.reset();
                    }

                    @Override
                    public void onAnimationEnd(Animator animation) {
                        super.onAnimationEnd(animation);
                        mShadowMatrix.reset();
                    }
                });
                mValueAnimator.setRepeatCount(mRepeatCount);
            }
        }
    }

```
- 简要说明几个重要变量：
	- `mShadowMatrix`,用来控制`Shader`位置的`Matrix`。
	- `mLinearGradient`,实现*闪光*效果的`Shader`，水平渐变层。
	- `mValueAnimator`，属性动画引擎。

##### 这里面使用了一个`LinearGradient `线性渐变的着色器。着重说一下它的使用方法。

先看一下`LinearGradient`的构造函数：

```
 /** Create a shader that draws a linear gradient along a line. 
        @param x0           The x-coordinate for the start of the gradient line 
        @param y0           The y-coordinate for the start of the gradient line 
        @param x1           The x-coordinate for the end of the gradient line 
        @param y1           The y-coordinate for the end of the gradient line 
        @param  colors      The colors to be distributed along the gradient line 
        @param  positions   May be null. The relative positions [0..1] of 
                            each corresponding color in the colors array. If this is null, 
                            the the colors are distributed evenly along the gradient line. 
        @param  tile        The Shader tiling mode 
    */  
    public LinearGradient(float x0, float y0, float x1, float y1, int colors[], float positions[],  
            TileMode tile) {  
.........  
.....  
......  
        }
```
>这其中：<br>x0，y0---->代表起点的坐标<br>x1，y1---->代表终点的坐标<br>colors---->colors表示渲染的颜色，它是一个颜色数组，数组长度必须大于等于2<br>positions---->positions表示colors数组中几个颜色的相对位置，是一个float类型的数组，该数组的长度必须与colors数组的长度相同。如果这个参数使用null也可以，这时系统会按照梯度线来均匀分配colors数组中的颜色<br>title---->代表了系统提供的几种渲染模式，这里选用了`LinearGradient.TileMode.CLAMP`模式，**表示重复colors数组里的最后一种颜色直到该View结束的地方**

*这里我们来做个试验，将colors的最后一个色值改为`Color.BLUE`*

```
mLinearGradient = new LinearGradient(0, 0, SHADOW_W, 0, new int[] {currentTextColor, Color.GREEN, Color.BLUE},
```
>OK,看一下试验效果.

![](https://wx4.sinaimg.cn/mw690/a28b91d8gy1fqunw73pdrg207i07qglz.gif)

**可见设置了`ClAMP`模式后，的确是将colors最后的一个色值覆盖到了未渲染区域**

#### 监听属性动画的数值更新，来触发重绘，`OnDraw（）`方法实现尤为简单。

```
@Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        Log.d(TAG, "onDraw: " + System.currentTimeMillis());
        if (mLinearGradient != null) {
            mLinearGradient.setLocalMatrix(mShadowMatrix);
        }
    }
```
- `Matrix`，改变位置即可实现*光芒*滑过效果

## 以上就是本次动画的简要原理阐述，项目地址在这里，个中原理已在注释上写的明明白白。[FlickerTextView](https://github.com/JadynAi/LoadingLovely/blob/master/app/src/main/java/com/example/jadynai/loadinglovely/flicker/TextFlickerView.java),大家喜欢的话不妨点个赞吧。

**基于需求的特殊性，本地动画使用了`TextView`。其实大可不必拘泥于此，本质上还是对`Paint`设置`Shader`的使用而已。**

**原创不易，大家走过路过看的开心，可以适当给个一毛两毛聊表心意**<br>![](http://JadynAi.github.io/img/person_wechat.jpg)



