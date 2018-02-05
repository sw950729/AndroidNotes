### Activity是什么？

Activity是用户和应用程序交互的界面，用户可以在Activity上进行点击、滚动、触摸等操作。一般来说，一个应用是由多个Activity组成，首次进入的Activity称为主Activity。至于如何判断一个Activity是不是主Activity。本篇文章我们先不讨论。后面会讲到。



### Activity的活动状态

当我查阅关于Activity的官方文档的时候，我发现，官方文档中谈到Activity有三种状态，运行中、暂停、停止。我觉得少了一个销毁。所以这边我简单介绍一下Activity的四种活动状态。

- 运行中(running)  

  此acitvity位于前台，并且用户可以在activity中执行触摸、点击、滚动等操作。

- 暂停(paused)

  activity可见，但并不可以操作，比如，当一个弹窗弹出来的时候。

- 停止(stopped)

  activity不可见，一般来说当用户按了home键之后。如果系统内存不够的时候，并且其他应用需要内存时，系统会回收已经停止的activity。

- 销毁(destroy)

  activity不可见，一般处于这种状态的activity会被系统回收掉。

  ​

### Activity的生命周期

首先我们先看下图解是怎么描述activity的生命周期的。

![](https://developer.android.google.cn/images/activity_lifecycle.png)

在正常情况下，Acitivity的生命周期会先后经历如下的生命周期：

1）onCreate：表示Activity正在被创建，一般来说，我们会在这个方法中设置布局以及一些数据的初始化。

2）onStart：表示Activity正在被启动，这个Activity已经可见，但并不是前台，这种情况下，我们无法操作这个Activity。

3）onResume：表示Activity已经可见并且位于前台了，此时，我们可以对当前的Activity进行操作。

4）onPasue：表示Activity暂停了，一般来说，执行到了这一步，后续会调用onStop方法。所以，在这边我们尽可能不要去执行耗时操作，因为这会影响新的Activity的显示。

5）onStop：表示Activity停止了，这边我们可以对一些数据进行回收。同样不能太耗时。

6）onDestroy：表示Activity即将销毁，这是Activity最后一个生命周期，同样，我们可以对一些数据进行回收。

7）onRestart：表示Activity正在被重启，这个方法一般只有在重启Activity的时候才被调用，比如，打开一个新的Activity然后在回退到当前的Activity。此方法便会被调用。



我们可以通过上述介绍以及图片，可以分析出一般Activity会有如下几种情况：

启动一个Activity：onCreate()-->onStart()-->onResume()。

按Home键之后或者重新打开一个Activity：onPause()--> onStop()（注：如果新的Activity是透明主题，当前Acitvity不会走onStop方法，例如dialog弹出的时候，当然，dialog关闭当前Activity会走onResume()）。

重新启动一个Activity：onRestart()-->onStart()-->onResume()。

销毁一个Activity：onPasue()-->onStop()-->onDestroy()。



### 异常情况下Activity的生命周期

首先，我们需要什么情况算异常情况。最好的例子莫过于横竖屏切换，这种情况Activity会被销毁并且重建，一般来说，系统会调用onSaveInstanceState 方法进行数据的存储。然后通过调用onRestoreInstanceState进行数据的恢复。如果我们想知道当前activity是否被重建了。我们可以在onCreate中判断bundle是否为null或者重写onRestoreInstanceState方法。一般推荐使用后者，因为只有在重建的情况下，此方法才会被调用。

如果我们希望当系统配置参数发生改变时，当前Activity不会被重建应该怎么做呢？自然是有的，我们可以在Mainfest文件中为Activity配置configChanges属性。比如，我们想当屏幕切换的时候当前Activity不被重启。我们可以这样：

```
android:configChanges="orientation|screenSize"
```

configChanges的选项有很多，这里我们常用的只有locale、orentation、keyboardHidden以及screenSize。（需要注意的是：screenSize这个属性比较特殊，minSdkVersion 和targetSdkVersion 低于13时，此选项不会导致activity重启。否则会导致activity重启。）

此时，我们可以通过重写此方法来进行一些特殊处理：

```
    @Override
    public void onConfigurationChanged(Configuration newConfig) {
        super.onConfigurationChanged(newConfig);
    }
```



### Thanks

《Android开发艺术探索》

Android官方API文档



