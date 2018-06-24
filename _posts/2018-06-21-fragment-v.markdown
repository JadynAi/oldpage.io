---
layout:     post
title:      "提供一种Fragment可见性改变的监测方案"
subtitle:   "Fragment相关"
date:       2018-06-21
author:     "JadynAi"
header-img: "img/post-bg-2015.jpg"
tags:
    - 技术讨论
---
**原创文章，转载请联系作者**
### 前言
Fragment，这个让人又爱又恨“碎片”。<br>使用它可以让项目更加轻便--我们可以将功能分割、复用，但其复杂的生命周期和`Transaction事务`，在极端操作【某些测试人员有一手绝活，三指甚至六指同时触屏乱弹】下会出现一些不可预期的错误--`Fragment`嵌套`Fragment`,横竖屏切换等等。<br>但无论怎样，面对解决问题，才是关键。这篇文章就是针对`Fragment`监测可见状态改变，提供一种解决方案。

### Fragment可见性解析
首先，要说明一下，这里的可见性就是对用户来说看的见。`嗯……肯定有人说这不废话吗？还用你来解释“可见”这个名词吗？`那么我来解释一下这其中的定义分歧。<br>一般来说，当APP的界面可见时，有些会理解成这个界面就位于顶层。但不排除另外一种情况，就是这个界面之上，“飘着”一个对话框或是半透明界面。那么这种情况下，由于其依然对用户可见，所以我会将其判定为`visible`。<br>**接下来会分析在特定交互环境下，Fragment内部被触发的方法。**

#### onResume
`Fragment`是不能单独存在的，它所在的视图树中，往下追溯，根部一定是一个`Activity`。在源码中，`onResume()`方法的描述很有意思。

```
/**
     * Called when the fragment is visible to the user and actively running.
     * This is generally
     * tied to {@link Activity#onResume() Activity.onResume} of the containing
     * Activity's lifecycle.
     */
    @CallSuper
    public void onResume() {
        mCalled = true;
    }
```
>一般情况下，对用户可见时触发。绑定在依赖的Activity生命周期里

也就是说，一般这个方法，会在可见并且正在活跃时被调用。但说到底，还是个“窝里造”，生命周期完全依赖于`父容器`----也一定依赖于**根Activity**。<br>那么不一般的情况下呢？<br>有这么一个例子，在进入一个`Activity`界面时，直接调用了`beginTransaction().hide(Fragment)`方法。那么一开始用户就不会看到这个界面，但生命周期确实也走到了`onResume`。此时可见性的判断，就不能依赖于这个方法。<br>这个交互下，走的是另外一个方法，请看下个小标题。

#### onHiddenChanged
这个方法在使用`beginTransaction().hide(Fragment)`会被调用，而且是在`onResume`之前。<br>先来看看源码里的描述。

```
 /* @param hidden True if the fragment is now hidden, false otherwise.
     */
    public void onHiddenChanged(boolean hidden) {
    }
```
>这个方法会回调出来一个参数，true的时候表示隐藏了，false表示可见。在可见性改变时被调用。<br>这里要注意一下这个布尔值的定义！

#### setUserVisibleHint
ViewPager搭配Fragment，也是常见的交互模式了。此时左右滑动时，这个方法会被触发。但有一点要说明一下，当ViewPager初始化时，Fragment相应的生命周期里。`setUserVisibleHint`方法是走在Fragment的`onCreate`之前的。

**以上几个方法，就是常见的交互下，会被触发的方法了。可见性的监测，主要也依赖于这个方法的相互配合。<br>这里还需要说明一下，可见性的监测，监测的是*“改变”*。也就是当Fragment被创建出来时，不会触发监测方法，不管它是可见还是不可见的状态。**

### 代码实现
在BaseFragment内，提供了一个`onVisibleToUserChanged(boolean isVisibleToUser)`方法作为内部回调。参数`isVisibleToUser`如字面所示，**True**表示可见，**false**不可见。当你需要在界面不可见，取消网络请求或是释放一些东西，你就可以使用此方案。<br>代码实现相当简单，就是一连串逻辑代码而已。只是在`onResume`方法里，需要判断一下是否已经触发了`onHiddenChanged`或是`setuserVisibleHint`方法。<br>代码很短，不到100行。这里直接贴出来。不方便的小可爱们，可以直接去[GitHub地址](https://github.com/JadynAi/KotlinDiary/blob/master/app/src/main/java/com/motong/cm/kotlintest/BaseFragment.kt).如果你喜欢的话，不妨点个赞吧。

```
abstract class BaseFragment : Fragment(){
    lateinit var mRootView: View
    private var isVisibleToUsers = false
    private var isOnCreateView = false
    private var isSetUserVisibleHint = false
    private var isHiddenChanged = false
    private var isFirstResume = false
    
    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        isOnCreateView = true
        mRootView = LayoutInflater.from(activity).inflate(getResId(), null, false)
        return mRootView
    }

    abstract fun getResId(): Int

    override fun onResume() {
        super.onResume()
        if (!isHiddenChanged && !isSetUserVisibleHint) {
            if (isFirstResume) {
                setVisibleToUser(true)
            }
        }
        if (isSetUserVisibleHint || (!isFirstResume && !isHiddenChanged)) {
            isVisibleToUsers = true
        }
        isFirstResume = true
    }

    override fun onPause() {
        super.onPause()
        isHiddenChanged = false
        isSetUserVisibleHint = false
        setVisibleToUser(false)
    }
    
    override fun setUserVisibleHint(isVisibleToUser: Boolean) {
        super.setUserVisibleHint(isVisibleToUser)
        isSetUserVisibleHint = true
        setVisibleToUser(isVisibleToUser)
    }

    override fun onHiddenChanged(hidden: Boolean) {
        super.onHiddenChanged(hidden)
        isHiddenChanged = true
        setVisibleToUser(!hidden)
    }
    
    private fun setVisibleToUser(isVisibleToUser: Boolean) {
        if (!isOnCreateView) {
            return
        }
        if (isVisibleToUser == isVisibleToUsers) {
            return
        }
        isVisibleToUsers = isVisibleToUser
        onVisibleToUserChanged(isVisibleToUsers)
    }

    protected open fun onVisibleToUserChanged(isVisibleToUser: Boolean) {
    }
}
```

### 结语
以上<br>![](http://JadynAi.github.io/img/wechat_official.png)
