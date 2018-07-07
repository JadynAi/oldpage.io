---
layout:     post
title:      "使用DSL模式构建Recyclerview适配器"
subtitle:   "Kotlin实践日记"
date:       2018-07-05
author:     "JadynAi"
header-img: "img/post-bg-2015.jpg"
tags:
    - RecyclerView
    - Kotlin
---

**原创文章，转载请联系作者**
### 前言
这是**Kotlin实践日记**的第一章，使用Kotlin构建一个，使用方便、多功能的Recyclerview适配器——`AcrobatAdapter`。<br>

不用写ViewHolder，也不用再特意为Recyclerview的Item添加点击事件。`AcrobatAdapter`让开发者专注于配置Item的UI和数据显示。仅仅需要几行有效代码，就能成功构建一个列表。

`AcrobatAdapter`的设计灵感来自于[夜叉](https://github.com/ssseasonnn/Yaksa)，我的函数名称也沿用**夜叉**的函数名，因为`itemDSL`这个名称是在太贴切了。夜叉这个项目是从另一个角度去构建Recyclerview，推荐小可爱们去看看。

### 使用文档

####普通列表

```
val acrobatAdapter = AcrobatAdapter<Int> {
          itemDSL {
              resId(R.layout.item_test)
              showItem { d, pos, view ->
                  view.item_tv.text = "数据Item" + d
              }
          }
       }.setData(数据源)
recycler_view.adapter = acrobatAdapter
```
>哒哒，以上代码就成功构建了一个列表<br>resId()函数绑定Item的布局ID<br>showItem函数内，渲染数据，d表示此项Item对应数据，会根据适配器泛型自动转换。pos为Item的position，view是Item的布局View。而且Kotlin支持使用Id代表控件，再也不用写findViewById啦！<br>最后，适配器的setData方法内置了DiffUtils，会对比新旧数据，这样在数据刷新时Recyclerview会有默认动画展示。让交互更加平滑。

#### 如果开发者在项目中，有好几个界面的Item完全一样，那岂不是还要写几套代码？完全不用担心，Item也支持复用。

首先Item相关类，继承`AcrobatItem`。

```
class Test : AcrobatItem<Int>() {
    override fun showItem(d: Int, pos: Int, view: View) {
        view.item_tv.text = "共用Item" + d
    }
    override fun getResId(): Int = R.layout.item_test
}
```
在适配器中：

```
val acrobatAdapter = AcrobatAdapter<Int> {
            item { 
                Test()
            }
        }.setData(数据源)
        recycler_view.adapter = acrobatAdapter
```

>让你的Item继承`AcrobatItem`即可。这样再多的界面复用Item也完全可行，但要注意的一点是：Item的数据类型必须一致。

#### 多Item样式可以咩？当然可以啦！

多item样式的话，写多个ItemDSL即可！每一个`ItemDSL`就代表一种独有的Item样式。同理，每调用一次`item`，也就多一种Item样式。

```
val acrobatAdapter = AcrobatAdapter<Int> {
       itemDSL {
            resId(R.layout.item_test)
            showItem { d, pos, view ->
                 view.item_tv.text = "数据Item: " + d
              }
              isMeetData { d, pos -> pos == 1 }
           }
        itemDSL {
             resId(R.layout.item_test1)
             showItem { d, pos, view ->
                  view.item_tv.text = "cece: " + d
              }
              isMeetData { d, pos -> pos != 1 }
           }
        }.setData(数据源)
  recycler_view.adapter = acrobatAdapter
```

>isMeetData函数两个参数为数据和position。用这两个参数来判断此Item在哪个位置、什么条件展示。譬如示例代码，position为1时是一个样式，不为1时是另一种样式。但要注意，所有的Item的isMeetData的条件都是互斥的噢。否则会抛出异常。

#### 嗯……Recyclerview没有默认的Item点击事件怎么办？没问题，`AcrobatAdapter`替你搞定。每个Item不但有单击`click`,还附带了双击`DoubleTap`和长按`LongPress`事件。而且完全不会影响Item布局View内部childView的事件。

- 每种Item都可以绑定自己独有的三个事件

```
val acrobatAdapter = AcrobatAdapter<Int> {
         itemDSL {
             resId(R.layout.item_test)
             showItem { d, pos, view ->
                 view.item_tv.text = "数据Item: " + d
             }
             onClick { 
                 toastS("单击")
             }
             onDoubleTap { 
                 toastS("双击")
             }
             
             longPress { 
                 toastS("长按")
             }
         }
         
         itemDSL {
             resId(R.layout.item_test1)
             showItem { d, pos, view ->
                 view.item_tv1.text = "另一种样式" + d
             }
             isMeetData { d, pos -> pos == 1 }
             
             onClick { 
                 toastS("单击另一种Item")
             }

             onDoubleTap {
                 toastS("双击另一种Item")
             }

             longPress {
                 toastS("长按另一种Item")
             }
         }         
       }.setData(数据源)
```
> 三个事件都是单独绑定Item的样式的！当使用多Item样式列表时，再也不用在click事件中，写很多的条件判断了！

#### `AcrobatAdapter`不但支持Item绑定事件，也支持Adapter外部绑定事件。但这样就需要开发者，在外部事件里区分多Item样式了。

- 适配器持有Item事件，使用`AcrobarAdapter`的`bindEvent()`函数

```
val acrobatAdapter = AcrobatAdapter<Int> {
         itemDSL {
             resId(R.layout.item_test)
             showItem { d, pos, view ->
                 view.item_tv.text = "数据Item: " + d
             }
             isMeetData { d, pos -> pos != 1 }
         }
        }.setData(data).bindEvent { 
            onClick { 
                toastS("外部单击")
            }
            
            onDoubleTap { 
                toastS("外部双击")
            }
            
            longPress { 
                toastS("外部长按")
            }
        }
```

#### 还有几个使用的小Tip

- ItemDSl的`onViewCreated(parent,view)`函数

```
val acrobatAdapter = AcrobatAdapter<Int> {
        itemDSL {
            resId(R.layout.item_test)
            showItem { d, pos, view ->
                view.item_tv.text = "数据Item: " + d
            }
            
            onViewCreate { parent, view -> 
                作一些和数据无关的UI操作，譬如view设置为圆形
                或者EditText的addTextChangedListener
            }
        }
      }
```
>showItem()函数是绑定在适配器的onBingViewHolder函数里的，触发会比较频繁。如果在`showItem`内做一些UI操作，会比较浪费性能.<br>`onViewCreated `函数是绑定在适配器的`onCreateViewHolder`函数内

- 刷新单个Item布局

Recyclerview如果需要刷新Item的话，不建议使用`notifyItemChanged(int position)`方法，因为这个方法会刷新整个Item的视图。在视觉上的直观体现就是，Item会闪烁。所以建议使用如下方法刷新Item：

```
notifyItemChanged(int position, Object payload)
```
使用上面这个方法，不会重绘整个View的视图。

```
val acrobatAdapter = AcrobatAdapter<Int> {
       itemDSL {
           resId(R.layout.item_test)
           showItem { d, pos, view ->
               view.item_tv.text = "数据Item: " + d
           }
           showItemPayload { d, pos, view, payloads -> 
            刷新Item的某个特定的ChildView。
            譬如在某个Item刷新进度条。下面为伪代码
            view.progreess_bar.setProgress(100%)        
           }
        }
     }
```
下面简单做一下效果展示：

#####使用`notifyItemChanged(int position)`和`showItem`刷新布局
![](https://mmbiz.qpic.cn/mmbiz_gif/jqw9LvhdsxJRdy8tPr5s35tNYfwkbEefz7II2pWngbRZmXNPibrFibJewMZbTicO7nlRiaTianeO3n7wfBhOElO8rAg/0?wx_fmt=gif)

#####使用`notifyItemChanged(int position, Object payload)`和`showItemPayload `刷新布局
![](https://mmbiz.qpic.cn/mmbiz_gif/jqw9LvhdsxJRdy8tPr5s35tNYfwkbEefhQdicXhluHW5fhETuP82kfMUBcvHdIPMy3xEPpeibCo5j0wtmtI25B9A/0?wx_fmt=gif)

效果对比，一目了然。


### 结语
Kotlin已成为Android开发的官方语言，不管工作上用得到用不到，大家了解一二还是有必要的。毕竟这个时代变化的太快了。<br>**以上**<br>![](http://JadynAi.github.io/img/wechat_official.png)


