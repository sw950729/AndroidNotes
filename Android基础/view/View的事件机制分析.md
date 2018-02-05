### 为什么要去分析view的事件

关于view的事件，百度google一搜。一批又一批。但是能让人理解的少之又少。换句话说，不是那些作者不懂。只是说，他懂了，但他讲解后不一定能让别人看得懂。我记得有人问我当初是怎么接触自定义view这东西的。因为他们觉得自定义view这个东西很难。我就回了如下几句话：自定义view你把paint和canvas。弄懂了基本也就差不多了。我这边说的是差不多，不是完全，你们别曲解哈= =当然前提是数学和物理要好= =。对于View来说，我认为，paint和canvas都不是重点，如何分析他的事件处理才是重点。下面我们一步步的来了解。

### View的结构

想要了解view的事件，他的结构我们是需要知道的，我们先放一张view的结构图。然后根据图来一步步分析：
[![这里写图片描述](http://img.blog.csdn.net/20170803153321208?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)](http://img.blog.csdn.net/20170803153321208?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
最顶层的PhoneWindow是什么呢？通过字面意思，我们知道他是手机窗口。我们需要知道window到底是什么才能分析phonewindow的作用。

简单来说，Window是一个抽象类，是所有视图的最顶层容器，视图的外观和行为都归他管，不论是背景显示，标题栏还是事件处理都是他管理的范畴，虽然能管的事情看似很多，但是没实权，因为抽象类不能直接使用。

而 PhoneWindow 作为 Window 的唯一实现类，PhoneWindow 的权利可是非常大大，不过对于我们来说用处并不大。

下面我们来说说DecorView：
这样，我们做个假设，一个跟布局高度是wrap的控件，我们运行之后可以发现，除了那个控件，留下了大量的空白区域，由于我们的手机屏幕不能透明，所以这些空白区域肯定要显示一些东西，那么应该显示什么呢？当然如果你没有设置全屏。我们还会发现标题栏状态栏那些。而这些就是所谓的DecorView。

### 事件处理的过程

我们先看下下面的表格：

| 类型   | 相关方法                  | Activity | ViewGroup | View |
| ---- | --------------------- | -------- | --------- | ---- |
| 事件分发 | dispatchTouchEvent    | √        | √         | √    |
| 事件拦截 | onInterceptTouchEvent | X        | √         | X    |
| 事件消费 | onTouchEvent          | √        | √         | √    |

从上表可以看到 Activity 和 View 都是没有事件拦截的，这是因为：

```
1. Activity 作为原始的事件分发者，如果 Activity 拦截了事件会导致整个屏幕都无法响应事件，这肯定不是我们想要的效果。
2. View最为事件传递的最末端，要么消费掉事件，要么不处理进行回传，根本没必要进行事件拦截。

```

所以我们知道了activity是最上层，而view是最底层，那么结合之前view的结构的那张图，我们可以知道view的传递流程应该是这样：

```
Activity －> PhoneWindow －> DecorView －> ViewGroup －> ... －> View
```

而view的处理恰恰相反，那就是这样:

```
Activity <－ PhoneWindow <－ DecorView <－ ViewGroup <－ ... <－ View
```

如果上面的你理解了，下面这张图，对你来说也就是小意思了。
[![这里写图片描述](http://img.blog.csdn.net/20170803160330745?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)](http://img.blog.csdn.net/20170803160330745?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

如果你已经完全理解了上面的内容。那么对于view的事件处理。你已经懂了50%了。

### 事件分发和拦截

事件分发和拦截的流程其实就是上面那张图。《群英传》中举得例子很恰当。我们现在也对所有的流程对应一个职位:

```
Activity:公司大boss
RootView：项目经理
ViewGroup：技术组长
View：码农
```

前面已经知道了处理流程，我们这边在写一次。
**分发(dispatch)–>拦截(onIntercept)–>消费(ontouch)**

**public boolean dispatchTouchEvent(MotionEvent ev)**

用来进行事件分发。如果事件能够传递到当前view。那么此方法一定会调用，返回结果受当前View的ontouch和下级的dispatchtouchevent影响，表示是否消耗当前事件。

**public boolean onInterceptTouchEvent(MotionEvent ev)**

在上述方法内部调用，用来判断是否拦截事件，如果当前view拦截了事件，那么在同一序列中此方法不会在调用，返回结果表示是否拦截事件。

**public boolean onTouchEvent(MotionEvent event)**

在dispatch方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，那么当前view再也无法接受事件。

| 类型   | true     | false  | 方法          |
| ---- | -------- | ------ | ----------- |
| 事件传递 | 拦截，不继续   | 不拦截，继续 | onIntercept |
| 事件处理 | 处理了，无需审核 | 给上级处理  | ontouch     |

默认情况下，我们都是返回false。

#### 默认的点击事件

我们先看下整体的效果图，再来进行点击查看结果：
[![这里写图片描述](http://img.blog.csdn.net/20170803163244742?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)](http://img.blog.csdn.net/20170803163244742?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

当然这是一个framelayout。你们只要知道是一层盖一层就对了= = 。那么现在所有的事件我们直接return super来看下效果图：
[![这里写图片描述](http://img.blog.csdn.net/20170803164453908?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)](http://img.blog.csdn.net/20170803164453908?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

我们通过上面列举的职位可以分析下情况，具体详情如下：

```
MainActivity [老板]: dispatchTouchEvent     经理,我准备发展一下电商业务,下周之前做一个淘宝出来.
RootView     [经理]: dispatchTouchEvent     呼叫技术部,老板要做淘宝,下周上线.
RootView     [经理]: onInterceptTouchEvent  (老板可能疯了,但又不是我做.)
ViewGroupA   [组长]: dispatchTouchEvent     老板要做淘宝,下周上线?
ViewGroupA   [组长]: onInterceptTouchEvent  (看着不太靠谱,先问问小王怎么看)
View1        [码农]: dispatchTouchEvent     做淘宝???
View1        [码农]: onTouchEvent           这个真心做不了啊.
ViewGroupA   [组长]: onTouchEvent           小王说做不了.
RootView     [经理]: onTouchEvent           报告老板, 技术部说做不了.
MainActivity [老板]: onTouchEvent           这么简单都做不了,你们都是干啥的(愤怒).
```

可以从log日志中看出，和我们上面的流程是一模一样的。我们尝试下对某个view的TouchEvent 直接拦截了（return true），然后在看看效果图：
[![这里写图片描述](http://img.blog.csdn.net/20170804094215210?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)](http://img.blog.csdn.net/20170804094215210?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
流程应该就是这样：

```
MainActivity [老板]: dispatchTouchEvent     把按钮做的好看一点,要有光泽,给人一种点击的欲望.
RootView     [经理]: dispatchTouchEvent     技术部,老板说按钮不好看,要加一道光.
RootView     [经理]: onInterceptTouchEvent  
ViewGroupA   [组长]: dispatchTouchEvent     给按钮加上一道光.
ViewGroupA   [组长]: onInterceptTouchEvent  
View1        [码农]: dispatchTouchEvent     加一道光.
View1        [码农]: onTouchEvent           做好了.
```

那么我们如果ViewGroup进行拦截处理呢？也就是View的事件不响应了。我们再一次的看看效果图：
[![这里写图片描述](http://img.blog.csdn.net/20170804095231990?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)](http://img.blog.csdn.net/20170804095231990?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

流程如下：

```
MainActivity [老板]: dispatchTouchEvent     现在项目做到什么程度了?
RootView     [经理]: dispatchTouchEvent     技术部,你们的app快做完了么?
RootView     [经理]: onInterceptTouchEvent  
ViewGroupA   [组长]: dispatchTouchEvent     项目进度?
ViewGroupA   [组长]: onInterceptTouchEvent  
ViewGroupA   [组长]: onTouchEvent           正在测试,明天就测试完了
```

### 总结

综上所失，我们可以发现，我们几乎没有在dispatch里面进行任何处理，不是说用不到，只是这个方法用的不多，一般都在拦截和消费里面进行处理，通知上层或者下层是否需要执行。昨天一个小伙伴刚和我说他最近也在重新整理view这块，也是一个功能就要用到了dispatch这个方法。这个具体我就不细说了。你们可以自己理理。

这个篇幅我只是对view做了一个简单的介绍，通过标题“浅析”也知道。不过，如果这篇你们看懂了。对于正常的需求开发已无大碍。后面我尽可能去深入分析一次。

### 参考文献

《android群英传》

[事件分发机制原理](http://www.gcssloop.com/customview/dispatch-touchevent-theory)

