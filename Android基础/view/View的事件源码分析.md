### 解析View源码

既然是分析源码，那么我们就要找准入手点，不然几万行代码看完在整理完还是很累的。既然是事件的分析，我们就应该知道从哪入手。就是touchevent了。

首先，我们先了解下下面几个属性，这肯定是和事件有关的。

1.clickable：控制当前view是否可以点击

2.longclickable：控制当前view是否可以长按

3.foucsable：是否可以获取当前view的焦点（一般用于edittext）

4.enable：是否可以选中

5.saveenable：状态保存，和activity的savedInstanceState类似

6.setOnTouchListener：触摸事件

7.setOnLongClickListener：长按事件

8.setClickListener：点击事件

上次我们讲过view的事件是从dispatchtouchevent–>onTouchEvent。这边我就不写自定义view了。普通一个view分别执行三个事件。我们先来看效果图：
[![这里写图片描述](http://img.blog.csdn.net/20170824160746366?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)](http://img.blog.csdn.net/20170824160746366?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### enable

首先我们先来了解最简单的enable属性，首先，我们先放上源码：

```
public void setEnabled(boolean enabled) {
        if (enabled == isEnabled()) return;
        setFlags(enabled ? ENABLED : DISABLED, ENABLED_MASK);
        /*
         * The View most likely has to change its appearance, so refresh
         * the drawable state.
         */
        refreshDrawableState();
        // Invalidate too, since the default behavior for views is to be
        // be drawn at 50% alpha rather than to change the drawable.
        invalidate(true);
        if (!enabled) {
            cancelPendingInputEvents();
        }
    }
```

我们直接看最后一个判断，当enable为false的我们发现他执行了一个方法，字面意思就是取消所有的输入事件。我们仔细看了他到底做了什么处理。

通过层层的调用，我们找到了如下代码：

```
public void onCancelPendingInputEvents() {
    removePerformClickCallback();
    cancelLongPress();
    mPrivateFlags3 |= PFLAG3_CALLED_SUPER;
}
```

里面分别执行了，移除所有的事件回调以及取消了长按操作。

于是我们便知道，只要调用这个方法，他的所有事件都将不会执行。

#### longclickable

我们把代码改成如下：

```
view.setLongClickable(false);
view.setOnTouchListener(new View.OnTouchListener() {
    @Override
    public boolean onTouch(View view, MotionEvent motionEvent) {
        if (motionEvent.getAction() == MotionEvent.ACTION_DOWN) {
            Log.i("-->", "onTouch: View--->onTouch");
        }
        return false;
    }
});

view.setOnLongClickListener(new View.OnLongClickListener() {
    @Override
    public boolean onLongClick(View view) {
        Log.i("-->", "onLongClick: View--->onLongClick");
        return false;
    }

});
view.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        Log.i("-->", "onClick: View--->onClick");
    }
});
```

然后我们在跑一下，发现了一个问题。
[![这里写图片描述](http://img.blog.csdn.net/20170824160746366?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)](http://img.blog.csdn.net/20170824160746366?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
我们把longclick事件都禁止了为什么还会执行长按事件呢，我们翻一下源码：

```
public void setOnLongClickListener(@Nullable OnLongClickListener l) {
       if (!isLongClickable()) {
           setLongClickable(true);
       }
       getListenerInfo().mOnLongClickListener = l;
   }
```

发现他的事件监听中先判断了是否长按如果不是，就强制把它变为ture，然后执行。然后我们往上看，可以发现，咦，onclick也是的。

```
public void setOnClickListener(@Nullable OnClickListener l) {
     if (!isClickable()) {
         setClickable(true);
     }
     getListenerInfo().mOnClickListener = l;
 }
```

所以我们需要在执行事件之后执行这些方法。然后我们在看一下效果：
[![这里写图片描述](http://img.blog.csdn.net/20170824160746366?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)](http://img.blog.csdn.net/20170824160746366?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

我们神器的发现，我们把longclickable方法onlongclick事件之后执行，效果依旧是这样。这是为什么呢？我们打开onTouchEvent的源码，看个究竟。

```
...
      final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE|| (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)|| (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;
     ...
  switch (action) {
  case MotionEvent.ACTION_UP:
    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
  if ((viewFlags & TOOLTIP) == TOOLTIP) {
     handleTooltipUp();
 }
     if (!clickable) {
      removeTapCallback();
      removeLongPressCallback();
      mInContextButtonPress = false;
      mHasPerformedLongPress = false;
      mIgnoreNextUpEvent = false;
       break;
       }
              ...
      if (mPerformClick == null) {
                   mPerformClick = new PerformClick();
         }
      if (!post(mPerformClick)) {
            performClick();
          }
      }
    }
   ...
```

源码太多，这边我省略了部分源码，留了几个重点，我们可以看下clickable是通过或的关系得到的，也就是只要长按和点击有一个执行，那他为ture。后面还有一段就是当clickable为false的时候移除所有的事件回调。

不过我们还发现了一个问题，onlongclick怎么没有执行，字面意思我们理解下，长按，那肯定是按下的时候执行的，我们来查找下：

```
case MotionEvent.ACTION_DOWN:
                  if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
                      mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
                  }
                  mHasPerformedLongPress = false;
                  if (!clickable) {
                      checkForLongClick(0, x, y);
                      break;
                  }
                  if (performButtonActionOnTouchDown(event)) {
                      break;
                  }
                  // Walk up the hierarchy to determine if we're inside a scrolling container.
                  boolean isInScrollingContainer = isInScrollingContainer();
                  // For views inside a scrolling container, delay the pressed feedback for
                  // a short period in case this is a scroll.
                  if (isInScrollingContainer) {
                      mPrivateFlags |= PFLAG_PREPRESSED;
                      if (mPendingCheckForTap == null) {
                          mPendingCheckForTap = new CheckForTap();
                      }
                      mPendingCheckForTap.x = event.getX();
                      mPendingCheckForTap.y = event.getY();
                      postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                  } else {
                      // Not inside a scrolling container, so show the feedback right away
                      setPressed(true, x, y);
                      checkForLongClick(0, x, y);
                  }
                  break;
```

这段代码其实就是判断，视图是否为短时间的滑动/滚动，如果不是的话，我们就去检查长按事件。我们来看下代码：

```
private void checkForLongClick(int delayOffset, float x, float y) {
     if ((mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE || (mViewFlags & TOOLTIP) == TOOLTIP) {
         mHasPerformedLongPress = false;
         if (mPendingCheckForLongPress == null) {
             mPendingCheckForLongPress = new CheckForLongPress();
         }
         mPendingCheckForLongPress.setAnchor(x, y);
         mPendingCheckForLongPress.rememberWindowAttachCount();
         mPendingCheckForLongPress.rememberPressedState();
         postDelayed(mPendingCheckForLongPress,
                 ViewConfiguration.getLongPressTimeout() - delayOffset);
     }
 }
```

和我们想象的一样，他会先检查longclickable然后在判断是否需要执行我们的长按事件。我们来把长按禁用看下效果：
[![这里写图片描述](http://img.blog.csdn.net/20170828132536062?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)](http://img.blog.csdn.net/20170828132536062?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3c5NTA3Mjk=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

#### dispatchTouchEvent

这样我们差不多把事件分析的源码整理的差不多了。不过呢，我们发现ontouchListener里面有一个事件，如果return true的话那么他将直接消耗掉事件，这个是如何处理的呢？我们去翻下源码，看看在哪边执行了这个方法。找呀找，嗯，找到了：

```
public boolean dispatchTouchEvent(MotionEvent event) {
        // If the event should be handled by accessibility focus first.
        if (event.isTargetAccessibilityFocus()) {
            // We don't have focus or no virtual descendant has it, do not handle the event.
            if (!isAccessibilityFocusedViewOrHost()) {
                return false;
            }
            // We have focus and got the event, then use normal event dispatch.
            event.setTargetAccessibilityFocus(false);
        }
        boolean result = false;
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(event, 0);
        }
        final int actionMasked = event.getActionMasked();
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Defensive cleanup for new gesture
            stopNestedScroll();
        }
        if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
        if (!result && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }
        // Clean up after nested scrolls if this is the end of a gesture;
        // also cancel it if we tried an ACTION_DOWN but we didn't want the rest
        // of the gesture.
        if (actionMasked == MotionEvent.ACTION_UP ||
                actionMasked == MotionEvent.ACTION_CANCEL ||
                (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
            stopNestedScroll();
        }
        return result;
    }
```

这个这个，源码似乎有点长哈。。不急，我来用几句通俗易懂的话来给你们讲清楚。

如果设置了OnTouchListener，并且当前 View 可点击，就调用监听器的 onTouch 方法， 如果 onTouch 方法返回值为 true，就设置 result 为 true。 如果 result 为 false，则调用自身的 onTouchEvent。如果 onTouchEvent 返回值为 true，则设置 result 为 true。

#### 总结

我们可以把View的事件分析总结成如下几句话：

1.view的事件可以理解成一个责任链模式，其实我当时就是因为了解了责任链模式，才会快速的理解view的事件传递的。

2.View的事件的调度顺序是 onTouchListener –> onTouchEvent –> onLongClickListener –> onClickListener 。

3.如果他的enable为false。那他将不执行任何事件，包括ontouch。

4.如果view的ontouch消耗了事件，他不再执行任何点击事件。

5.对于click的处理，如果想只执行longclick不执行的click的方法，只有选择不去监听click，至于为什么，我们前面分析过。