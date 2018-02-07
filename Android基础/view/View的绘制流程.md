### 概述

View的绘制流程主要是指测量、布局以及绘制显示，在View中，measure是测量View的宽高，layout是控制View四个顶点的位置，而draw就是将布局直接绘制出来。

### Measure流程

measure的流程氛围View的measure流程以及ViewGroup的measure的流程。之所以把View和ViewGroup分开就是因为ViewGroup不仅仅要测量自身的宽高，而且还需要通过递归将子view的宽高测量出来。

#### View的measure过程

View的measure说简单也简单，说复杂也复杂，我们直接通过源码来看看measure的时候到底经历了什么。
```
    public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
          ···
                int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
                if (cacheIndex < 0 || sIgnoreMeasureCache) {
                    // measure ourselves, this should set the measured dimension flag back
                    onMeasure(widthMeasureSpec, heightMeasureSpec);
                    mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
                } else {
                    long value = mMeasureCache.valueAt(cacheIndex);
                    // Casting a long to int drops the high 32 bits, no mask needed
                    setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                    mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
                }
    	···
        }
```
我们可以通过源码发现View的measure是final类型的，所以此方法无法被重写。但是他内部通过调用onMeasure进行了测量。那么我们只需要看onMeasure的具体实现即可。View的onMeasure代码如下：
```
       protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
            setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                    getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
        }
```
这个代码很少，但是少能代表着简单么？我们先看看setMeasuredDimension这个方法，从字面意思理解就是设置测量的值，那么我们看看源码呢：
```
     protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
            boolean optical = isLayoutModeOptical(this);
            if (optical != isLayoutModeOptical(mParent)) {
                Insets insets = getOpticalInsets();
                int opticalWidth  = insets.left + insets.right;
                int opticalHeight = insets.top  + insets.bottom;
    
                measuredWidth  += optical ? opticalWidth  : -opticalWidth;
                measuredHeight += optical ? opticalHeight : -opticalHeight;
            }
            setMeasuredDimensionRaw(measuredWidth, measuredHeight);
        }
```
如果你细看了measure的源码，你会发现，上面的代码在measure这个方法中出现过，并且我上面放出来的代码也设计到了setMeasuredDimensionRaw这个方法，说白了。不管你走不走onMeasure这个方法，它最后都会走setMeasuredDimensionRaw这个方法。也就是设置测量的宽和高。

这个方法看完了。我们回到onMeasure中在看看其他方法到底处理了什么？剩下的方法是getSuggestedMinimumWidth和getDefaultSize我们一个个的看，首先看看getSuggestedMinimumWidth中处理了什么：
```
        protected int getSuggestedMinimumWidth() {
            return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
        }
```
代码不多，也很好理解，首先判断它是否有背景，如果没有背景就取宽度的最小值，如果有背景，那我们就在背景的最小值和视图的最小值中取最大值，mBackground.getMinimumWidth()的返回值我们也可以通过源码发现最后返回的是0。这边是宽度的，那么高度同理，我这边就不分析了。

下面我们看getDefaultSize的源码是什么： 
```
     public static int getDefaultSize(int size, int measureSpec) {
            int result = size;
            int specMode = MeasureSpec.getMode(measureSpec);
            int specSize = MeasureSpec.getSize(measureSpec);
    
            switch (specMode) {
            case MeasureSpec.UNSPECIFIED:
                result = size;
                break;
            case MeasureSpec.AT_MOST:
            case MeasureSpec.EXACTLY:
                result = specSize;
                break;
            }
            return result;
        }
```
这个方法也不难，对于我们来说，我们只要看AT_MOST和EXACTLY这两个，其实这边返回的size也就相当于MeasureSpec中返回的size。这个size是测量后的大小，之所以是测量后的大小是因为View分成getMeasureWidth和getWidth2个方法，前者是在onMeasure之后拿到的，而后者是在layout方法之后拿到的。不过，一般情况下，这2个值是相等的。

#### ViewGroup的measure流程

在viewGroup没有onMeasure方法，但是有MeasureChildren方法。我们继续通过源码来查看下里面到底处理了什么：
```
        protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
            final int size = mChildrenCount;
            final View[] children = mChildren;
            for (int i = 0; i < size; ++i) {
                final View child = children[i];
                if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                    measureChild(child, widthMeasureSpec, heightMeasureSpec);
                }
            }
        }
```
可以看到，这边拿到子View的数量后，循环遍历通过measureChild将子View绘制进去。我们继续往下走看看measureChild处理了什么：
```
     protected void measureChild(View child, int parentWidthMeasureSpec,
                int parentHeightMeasureSpec) {
            final LayoutParams lp = child.getLayoutParams();
    
            final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                    mPaddingLeft + mPaddingRight, lp.width);
            final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                    mPaddingTop + mPaddingBottom, lp.height);
    
            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
```
它是通过拿到子view的布局参数，讲子view的宽度以及padding算进去测量。而我们继续往下看ViewGroup源码的时候，我们可以发现它还有一个方法就是measureChildWithMargins，字面意思就是测量的时候算上margins的值，具体我们还是通过源码来看一下：
```
     protected void measureChildWithMargins(View child,
                int parentWidthMeasureSpec, int widthUsed,
                int parentHeightMeasureSpec, int heightUsed) {
            final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    
            final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                    mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                            + widthUsed, lp.width);
            final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                    mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                            + heightUsed, lp.height);
    
            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
```
的确如此，综上，我们可以发现，ViewGroup的测量主要是通measureChild以及measureChildWithMargins来实现的。

### layout流程

layout这个方法是用来控制View的四个点显示的位置的，换句话说，这四个点确认了，它在父容器的位置也就确认了。我们先来看看View的layout方法：
```
       public void layout(int l, int t, int r, int b) {
            if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
                onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }
    
            int oldL = mLeft;
            int oldT = mTop;
            int oldB = mBottom;
            int oldR = mRight;
    
            boolean changed = isLayoutModeOptical(mParent) ?
                    setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    
            if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
                onLayout(changed, l, t, r, b);
                mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
    
                ListenerInfo li = mListenerInfo;
                if (li != null && li.mOnLayoutChangeListeners != null) {
                    ArrayList<OnLayoutChangeListener> listenersCopy =
                            (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                    int numListeners = listenersCopy.size();
                    for (int i = 0; i < numListeners; ++i) {
                        listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                    }
                }
            }
    
            mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
            mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
        }
```
它是通过setFrame来确定view的四个顶点的位置，layout和measure一样，是final类型的，无法被重写，我们需要通过实现onlayout方法来进行view的绘制，不过我们可以发现View和ViewGroup的onlayout方法中并没有真正实现。那我们就通过最常用的Linearlayout源码来分析onlayout方法。

#### LinearLayout的layout流程

首先，我们来看onlayout方法： 
```
       protected void onLayout(boolean changed, int l, int t, int r, int b) {
            if (mOrientation == VERTICAL) {
                layoutVertical(l, t, r, b);
            } else {
                layoutHorizontal(l, t, r, b);
            }
        }
```
可以发现，其内部通过判断显示样式来进行布局位置的控制，我们就拿layoutVertical来进行分析把：
```
        void layoutVertical(int left, int top, int right, int bottom) {
            ···
            final int count = getVirtualChildCount();
    		···
    
            for (int i = 0; i < count; i++) {
                final View child = getVirtualChildAt(i);
                if (child == null) {
                    childTop += measureNullChild(i);
                } else if (child.getVisibility() != GONE) {
                    final int childWidth = child.getMeasuredWidth();
                    final int childHeight = child.getMeasuredHeight();
                    
                    final LinearLayout.LayoutParams lp =
                            (LinearLayout.LayoutParams) child.getLayoutParams();
                    ···
                    
                    if (hasDividerBeforeChildAt(i)) {
                        childTop += mDividerHeight;
                    }
    
                    childTop += lp.topMargin;
                    setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                            childWidth, childHeight);
                    childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);
    
                    i += getChildrenSkipCount(child, i);
                }
            }
        }
```
我们可以发现此方法会遍历setChildFrame对子view的位置进行指定。每绘制一次，childTop都会随之增大，这就意味着后面的元素将会靠下放置，也刚好符合了linearlayout的vertical方法，setChildFrame此方法只是调用子view的layout方法，相当于在父view的onlayout方法中，子view拿到四个点的位置通过调用自己的layout方法，进行定位。

### Draw流程

对于绘制，相对前2个就很简单了，因为前面我们已经可以正确的拿到了布局的宽高以及布局的位置，现在我们需要通过draw方法将其绘制出来。我们来看看源码：
```
     public void draw(Canvas canvas) {
            final int privateFlags = mPrivateFlags;
            final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                    (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
            mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;
    
            /*
             * Draw traversal performs several drawing steps which must be executed
             * in the appropriate order:
             *
             *      1. Draw the background
             *      2. If necessary, save the canvas' layers to prepare for fading
             *      3. Draw view's content
             *      4. Draw children
             *      5. If necessary, draw the fading edges and restore layers
             *      6. Draw decorations (scrollbars for instance)
             */
    
            // Step 1, draw the background, if needed
            int saveCount;
    
            if (!dirtyOpaque) {
                drawBackground(canvas);
            }
    
            // skip step 2 & 5 if possible (common case)
            final int viewFlags = mViewFlags;
            boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
            boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
            if (!verticalEdges && !horizontalEdges) {
                // Step 3, draw the content
                if (!dirtyOpaque) onDraw(canvas);
    
                // Step 4, draw the children
                dispatchDraw(canvas);
    
                // Overlay is part of the content and draws beneath Foreground
                if (mOverlay != null && !mOverlay.isEmpty()) {
                    mOverlay.getOverlayView().dispatchDraw(canvas);
                }
    
                // Step 6, draw decorations (foreground, scrollbars)
                onDrawForeground(canvas);
    
                // we're done...
                return;
            }
    		···
        }
```
可以通过源码的注释发现，如果可能的话跳过第二步和第五步。那我们看看其他的，第一步：绘制背景，第三步：绘制内容，第四步：绘制子view。第六步：绘制装饰，例如，前景，滚动条等等。

当然在view中还有2个比较重要的方法，我这边稍微提一下，invalidate()和requestLayout()。调用invalidate方法只会执行onDraw方法；调用requestLayout方法只会执行onMeasure方法和onLayout方法，并不会执行onDraw方法。

所以当我们进行View更新时，若仅View的显示内容发生改变且新显示内容不影响View的大小、位置，则只需调用invalidate方法；若View宽高、位置发生改变且显示内容不变，只需调用requestLayout方法；若两者均发生改变，则需调用两者，按照View的绘制流程，推荐先调用requestLayout方法再调用invalidate方法。










