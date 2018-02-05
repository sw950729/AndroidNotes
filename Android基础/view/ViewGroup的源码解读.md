### 解读ViewGroup

我们都知道，一个事件完整的流程是从dispatchTouchevent–>onInterceptTouchevent–>onTouchEvent。我们先不说事件监听的问题。上述三个步骤就是正常一个点击的流程。前面我们分析view的时候发现它并没有onInterceptTouchevent这个方法。这个我之前有提到，view已经是最底层了，所以就不需要拦截了。而这一整套的机制就是在ViewGroup中体现出来的。我们先来看一张图：
[![这里写图片描述](https://github.com/LittleFriendsGroup/AndroidSdkSourceAnalysis/raw/master/article/images/viewgroup_touchevent.png)](https://github.com/LittleFriendsGroup/AndroidSdkSourceAnalysis/raw/master/article/images/viewgroup_touchevent.png)

触摸事件发生后，在Activity内最先接收到事件的是Activity自身的dispatchTouchEvent，然后Activity传递给Activity的Window。接着Window传递给最顶端的View，也就是DecorView。接下来才是我们熟悉的触摸事件流程：首先是最顶端的ViewGroup(这边便是DecorView)的dispatchTouchEvent接收到事件。并通过onInterceptTouchEvent判断是否需要拦截。如果拦截则分配到ViewGroup自身的onTouchEvent，如果不拦截则查找位于点击区域的子View(当事件是ACTION_DOWN的时候，会做一次查找并根据查找到的子View设定一个TouchTarget，有了TouchTarget以后，后续的对应id的事件如果不被拦截都会分发给这一个TouchTarget)。查找到子View以后则调用dispatchTransformedTouchEvent把MotionEvent的坐标转换到子View的坐标空间，这不仅仅是x，y的偏移，还包括根据子View自身矩阵的逆矩阵对坐标进行变换(这就是使用setTranslationX,setScaleX等方法调用后，子View的点击区域还能保持和自身绘制内容一致的原因。使用Animation做变换点击区域不同步是因为Animation使用的是Canvas的矩阵而不是View自身的矩阵来做变换)。

### dispatchTouchevent分析

我们先放上dispatchTouchevent的源码，然后一步一步来分析：

```
public boolean dispatchTouchEvent(MotionEvent ev) {
       if (mInputEventConsistencyVerifier != null) {
           mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
       }
       // If the event targets the accessibility focused view and this is it, start
       // normal event dispatch. Maybe a descendant is what will handle the click.
       if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
           ev.setTargetAccessibilityFocus(false);
       }
       boolean handled = false;
       if (onFilterTouchEventForSecurity(ev)) {
           final int action = ev.getAction();
           final int actionMasked = action & MotionEvent.ACTION_MASK;
           // Handle an initial down.
           if (actionMasked == MotionEvent.ACTION_DOWN) {
               // Throw away all previous state when starting a new touch gesture.
               // The framework may have dropped the up or cancel event for the previous gesture
               // due to an app switch, ANR, or some other state change.
               cancelAndClearTouchTargets(ev);
               resetTouchState();
           }
           // Check for interception.
           final boolean intercepted;
           if (actionMasked == MotionEvent.ACTION_DOWN
                   || mFirstTouchTarget != null) {
               final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
               if (!disallowIntercept) {
                   intercepted = onInterceptTouchEvent(ev);
                   ev.setAction(action); // restore action in case it was changed
               } else {
                   intercepted = false;
               }
           } else {
               // There are no touch targets and this action is not an initial down
               // so this view group continues to intercept touches.
               intercepted = true;
           }
           // If intercepted, start normal event dispatch. Also if there is already
           // a view that is handling the gesture, do normal event dispatch.
           if (intercepted || mFirstTouchTarget != null) {
               ev.setTargetAccessibilityFocus(false);
           }
           // Check for cancelation.
           final boolean canceled = resetCancelNextUpFlag(this)
                   || actionMasked == MotionEvent.ACTION_CANCEL;
           // Update list of touch targets for pointer down, if needed.
           final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
           TouchTarget newTouchTarget = null;
           boolean alreadyDispatchedToNewTouchTarget = false;
           if (!canceled && !intercepted) {
               // If the event is targeting accessiiblity focus we give it to the
               // view that has accessibility focus and if it does not handle it
               // we clear the flag and dispatch the event to all children as usual.
               // We are looking up the accessibility focused host to avoid keeping
               // state since these events are very rare.
               View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                       ? findChildWithAccessibilityFocus() : null;
               if (actionMasked == MotionEvent.ACTION_DOWN
                       || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                       || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                   final int actionIndex = ev.getActionIndex(); // always 0 for down
                   final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                           : TouchTarget.ALL_POINTER_IDS;
                   // Clean up earlier touch targets for this pointer id in case they
                   // have become out of sync.
                   removePointersFromTouchTargets(idBitsToAssign);
                   final int childrenCount = mChildrenCount;
                   if (newTouchTarget == null && childrenCount != 0) {
                       final float x = ev.getX(actionIndex);
                       final float y = ev.getY(actionIndex);
                       // Find a child that can receive the event.
                       // Scan children from front to back.
                       final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                       final boolean customOrder = preorderedList == null
                               && isChildrenDrawingOrderEnabled();
                       final View[] children = mChildren;
                       for (int i = childrenCount - 1; i >= 0; i--) {
                           final int childIndex = getAndVerifyPreorderedIndex(
                                   childrenCount, i, customOrder);
                           final View child = getAndVerifyPreorderedView(
                                   preorderedList, children, childIndex);
                           // If there is a view that has accessibility focus we want it
                           // to get the event first and if not handled we will perform a
                           // normal dispatch. We may do a double iteration but this is
                           // safer given the timeframe.
                           if (childWithAccessibilityFocus != null) {
                               if (childWithAccessibilityFocus != child) {
                                   continue;
                               }
                               childWithAccessibilityFocus = null;
                               i = childrenCount - 1;
                           }
                           if (!canViewReceivePointerEvents(child)
                                   || !isTransformedTouchPointInView(x, y, child, null)) {
                               ev.setTargetAccessibilityFocus(false);
                               continue;
                           }
                           newTouchTarget = getTouchTarget(child);
                           if (newTouchTarget != null) {
                               // Child is already receiving touch within its bounds.
                               // Give it the new pointer in addition to the ones it is handling.
                               newTouchTarget.pointerIdBits |= idBitsToAssign;
                               break;
                           }
                           resetCancelNextUpFlag(child);
                           if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                               // Child wants to receive touch within its bounds.
                               mLastTouchDownTime = ev.getDownTime();
                               if (preorderedList != null) {
                                   // childIndex points into presorted list, find original index
                                   for (int j = 0; j < childrenCount; j++) {
                                       if (children[childIndex] == mChildren[j]) {
                                           mLastTouchDownIndex = j;
                                           break;
                                       }
                                   }
                               } else {
                                   mLastTouchDownIndex = childIndex;
                               }
                               mLastTouchDownX = ev.getX();
                               mLastTouchDownY = ev.getY();
                               newTouchTarget = addTouchTarget(child, idBitsToAssign);
                               alreadyDispatchedToNewTouchTarget = true;
                               break;
                           }
                           // The accessibility focus didn't handle the event, so clear
                           // the flag and do a normal dispatch to all children.
                           ev.setTargetAccessibilityFocus(false);
                       }
                       if (preorderedList != null) preorderedList.clear();
                   }
                   if (newTouchTarget == null && mFirstTouchTarget != null) {
                       // Did not find a child to receive the event.
                       // Assign the pointer to the least recently added target.
                       newTouchTarget = mFirstTouchTarget;
                       while (newTouchTarget.next != null) {
                           newTouchTarget = newTouchTarget.next;
                       }
                       newTouchTarget.pointerIdBits |= idBitsToAssign;
                   }
               }
           }
           // Dispatch to touch targets.
           if (mFirstTouchTarget == null) {
               // No touch targets so treat this as an ordinary view.
               handled = dispatchTransformedTouchEvent(ev, canceled, null,
                       TouchTarget.ALL_POINTER_IDS);
           } else {
               // Dispatch to touch targets, excluding the new touch target if we already
               // dispatched to it.  Cancel touch targets if necessary.
               TouchTarget predecessor = null;
               TouchTarget target = mFirstTouchTarget;
               while (target != null) {
                   final TouchTarget next = target.next;
                   if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                       handled = true;
                   } else {
                       final boolean cancelChild = resetCancelNextUpFlag(target.child)
                               || intercepted;
                       if (dispatchTransformedTouchEvent(ev, cancelChild,
                               target.child, target.pointerIdBits)) {
                           handled = true;
                       }
                       if (cancelChild) {
                           if (predecessor == null) {
                               mFirstTouchTarget = next;
                           } else {
                               predecessor.next = next;
                           }
                           target.recycle();
                           target = next;
                           continue;
                       }
                   }
                   predecessor = target;
                   target = next;
               }
           }
           // Update list of touch targets for pointer up or cancel, if needed.
           if (canceled
                   || actionMasked == MotionEvent.ACTION_UP
                   || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
               resetTouchState();
           } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
               final int actionIndex = ev.getActionIndex();
               final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
               removePointersFromTouchTargets(idBitsToRemove);
           }
       }
       if (!handled && mInputEventConsistencyVerifier != null) {
           mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
       }
       return handled;
   }
```

是不是整个人都蒙蔽了，这么长一串。其实整段代码可以缩减成几句话，就是这样：

```
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean result = false;             
    if (!onInterceptTouchEvent(ev)) { 
        result = child.dispatchTouchEvent(ev);
    }

    if (!result) {                    
        result = onTouchEvent(ev);
    }

    return result;
}
```

默认不消耗事件，如果本身没有拦截，就交给子类的dispatch事件，如果事件没有消费，就调用自身的onTouchEvent事件。你们仔细想想，流程是不是这样的？

好了，我们现在开始分析整个dispatch事件。具体说明和代码，你们自己对应= =因为太长了。

**对action_down的处理：**
我们发现，刚进方法的时候有个判断，第一次按下的时候，他会通过 cancelAndClearTouchTargets(ev)取消并且清除所有的手势操作，并且通过resetTouchState()把手势状态设置成默认状态。

接下来的操作，当然就是检查是否需要拦截事件拉。既然是拦截，当然就会走onInterceptTouchEvent这个方法了。我们来看看，viewgroup的onInterceptTouchEvent方法是怎么处理的。

```
public boolean onInterceptTouchEvent(MotionEvent ev) {
       if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
               && ev.getAction() == MotionEvent.ACTION_DOWN
               && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
               && isOnScrollbarThumb(ev.getX(), ev.getY())) {
           return true;
       }
       return false;
   }
```

我们可以发现，他默认就是false的。那么我们继续回到dispatch看。判断是否拦截后，我们发现他还执行了一句话`ev.setAction(action)` 官方说明是恢复操作，防止被更改。

**事件处理**
接下来就是检查事件是否取消咯。如果没有取消并且没有拦截就执行正常的事件处理。

如果事件是针对可访问性焦点视图，我们将其提供给具有可访问性焦点的视图。如果它不处理它，我们清除该标志并像往常一样将事件分派给所有的 ChildView。我们检测并避免保持这种状态，因为这些事非常罕见。这段是官方的解释。我们继续向下看，他执行这样一个方法removePointersFromTouchTargets(idBitsToAssign)。是为了防止指针不同步，清除之前的触摸标识。自我认为可能会和多指触控有关，先不管他，我们继续向下分析。

接下来就是打造了，他会先得到触摸点的坐标位置，然后在当前位置查找可接触的ChildView。然后重点！！！他的查找顺序是从后向前查找。什么意思呢？就是如果A和B有重叠的部分，并且B在A的上面，那么他处理的便是B的事件了。而不处理A的事件。

如果子View可以接受事件，那么我们就给他一个触摸的标识。接下来他会通过调用dispatchTransformedTouchEvent把事件分配给子View。

最后他会判断是否有touchtarget。如果没有的话，那就处理子view的事件。否则就会遍历touchtarget处理事件，也就是之前说的多点触控。在往后就是对action_up和cancel做的一些处理了，譬如：重置手势状态，移除多指操作等等。

### dispatchTransformedTouchEvent分析

前面我们说到了，会通过这个方法把事件分发给子view。我们还是先来看代码：

```
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    final boolean handled;
    // Canceling motions is a special case.  We don't need to perform any transformations
    // or filtering.  The important part is the action, not the contents.
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        if (child == null) {
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }
    // Calculate the number of pointers to deliver.
    final int oldPointerIdBits = event.getPointerIdBits();
    final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;
    // If for some reason we ended up in an inconsistent state where it looks like we
    // might produce a motion event with no pointers in it, then drop the event.
    if (newPointerIdBits == 0) {
        return false;
    }
    // If the number of pointers is the same and we don't need to perform any fancy
    // irreversible transformations, then we can reuse the motion event for this
    // dispatch as long as we are careful to revert any changes we make.
    // Otherwise we need to make a copy.
    final MotionEvent transformedEvent;
    if (newPointerIdBits == oldPointerIdBits) {
        if (child == null || child.hasIdentityMatrix()) {
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                final float offsetX = mScrollX - child.mLeft;
                final float offsetY = mScrollY - child.mTop;
                event.offsetLocation(offsetX, offsetY);
                handled = child.dispatchTouchEvent(event);
                event.offsetLocation(-offsetX, -offsetY);
            }
            return handled;
        }
        transformedEvent = MotionEvent.obtain(event);
    } else {
        transformedEvent = event.split(newPointerIdBits);
    }
    // Perform any necessary transformations and dispatch.
    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }
        handled = child.dispatchTouchEvent(transformedEvent);
    }
    // Done.
    transformedEvent.recycle();
    return handled;
}
```

这段代码就比之前简单很多了。我们会发现，他先判断状态是否取消，如果取消了，把当前事件变成取消状态，然后在判断是否有子view。如果有子view的话直接调用子view的dispatch事件。下面就是多指了，一个pointer对应一个ID，防止处理冲突。我印象中能简单粗暴的处理多指，应该是ViewDragHelper了。具体，你们可以自己去看。后面就如之前一样，判断child是否为null。然后得到是执行自身的事件还是child的事件。

### 总结

1.ViewGroup包涵多个子view的时候，我们是从后遍历，判断当前view是否可以点击，然后分发给需要处理的子view。
2.我们可以在onInterceptTouchEvent中进行事件拦截。
3.我们可以发现ViewGroup没有onTouchEvent事件，说明他的处理逻辑和View是一样的。
4.子view如果消耗了事件，那么ViewGroup就不会在接受到事件了。

最后我放上[ViewGroup源码](https://android.googlesource.com/platform/frameworks/base/+/master/core/java/android/view/ViewGroup.java)，你们可以自己去了解。