# 理解Window和WindowManager
### 概述
Window表示一个窗口，但在日常开发中我们接触的不多。我们常见的如Toast和PopWindow都是属于Window。Window是一个抽象类，而Window的具体实现类是PhoneWindow。如果我们需要创建一个Window，只需要通过WindowManager去实现。而它的具体实现是在WindowManagerService中。我们需要知道Android所有的视图都是附加在Window上的，换句话说Window是View的直接管理者。

### Window与WindowManager
在分析Window得工作机制之前，我们先看看如何添加一个Window，具体的我们可以通过WindowManager去添加，实现代码如下：

```java
        TextView text = new TextView(this);
        text.setText("just a test");
        WindowManager.LayoutParams layoutParams = new WindowManager.LayoutParams(
                WindowManager.LayoutParams.WRAP_CONTENT,
                WindowManager.LayoutParams.WRAP_CONTENT,
                0, 0,
                PixelFormat.TRANSPARENT
        );
        // flag 设置 Window 属性
        layoutParams.flags= WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL;
        // type 设置 Window 类别（层级）
        layoutParams.type = WindowManager.LayoutParams.TYPE_SYSTEM_OVERLAY;
        layoutParams.gravity = Gravity.CENTER;
        WindowManager windowManager = getWindowManager();
        windowManager.addView(text, layoutParams);
```
代码中我们并没有通过setContentView去设置布局，而是直接通过WindowManager去addView实现的。同时，我们需要去添加一个窗口的权限：

```java
 <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
```

### Window的分类
在上述代码中，我们可以设置Window的级别。Window有三种级别，分别为：应用Window、系统Window和子Window。应用Window对应一个Activity，子Window必须要依附在一个父Window上。

| Window | 	层级 | 
| ------ | ------ |
| 应用 Window |1~99 |
| 子 Window | 1000~1999|
|系统Window|2000~2999|

我们可以通过WindowManager.LayoutParams的type去设置，如果设置系统Window则需要加上上面的权限，否则会出现异常。

### WindowManager的内部机制
在开发中，我们无法直接使用Window，所有对Window的操作都是经过WindowManager去操作的。WindowManager所提供的功能很简单，即添加View，删除View和更新View。这三个的方法是在ViewManager接口中。代码如下：

```java

public interface ViewManager{

    public void addView(View view, ViewGroup.LayoutParams params);
    
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    
    public void removeView(View view);
}
```
重上述代码可以看出，我们只需要传入对应的view以及view的layoutparms属性即可进行添加和更新，如果需要删除，只需传入需要删除的view即可。


### Window的内部机制
#### Window的添加过程
前面说到如果需要对Window操作都需要通过WindowManager去实现，不过WindowManager只是一个接口，而它的真正实现类是WindowManagerImpl，在WindowManagerImpl的三大操作代码如下：

```java
   @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }

    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }
    
      @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }

```
可以看出，WindowManagerImpl并没有去处理，而是交给WindowManagerGlobal去处理。WindowManagerGlobal的addView主要是如下几个步骤：

1、检查参数;如果是子Window就去做相应的调整：

```java
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }
        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        }
```
2、创建 ViewRootImpl 并将 View 添加到集合中：  
在WindowManagerGlobal中有如下几个集合：

```java 
    private final ArrayList<View> mViews = new ArrayList<View>();
    private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
    private final ArrayList<WindowManager.LayoutParams> mParams =
            new ArrayList<WindowManager.LayoutParams>();
    private final ArraySet<View> mDyingViews = new ArraySet<View>();
```
|集合|存储内容|
|---|---|
|mViews|Window 所对应的 View|
|mRoots|Window 所对应的 ViewRootImpl|
|mParams|Window 所对应的布局参数|
|mDyingViews|正在被删除还没完全移除的 View 对象|
addView 操作时会将相关对象添加到对应集合中：

```java
    root = new ViewRootImpl(view.getContext(), display);
    view.setLayoutParams(wparams);
    mViews.add(view);
    mRoots.add(root);
    mParams.add(wparams);
```

3、通过 ViewRootImpl 来更新界面并完成 Window 的添加过程：  
最后通过如下代码进行addView：

```java
   root.setView(view, wparams, panelParentView);
```
在setView内部，我们会通过requsetLayout去异步刷新UI，requsetLayout代码如下：

```java
  public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```
这是View的绘制入口，我们接着往下看，后面它会调用：

```java

    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                getHostVisibility(), mDisplay.getDisplayId(),
                mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                mAttachInfo.mOutsets, mInputChannel);
```
它是通过WindowSession去完成最后的添加。WindowSession是一个Binder对象，最终实现类是Session，我们继续看代码：

```java
 @Override
    public int addToDisplay(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets,
            Rect outOutsets, InputChannel outInputChannel) {
        return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
                outContentInsets, outStableInsets, outOutsets, outInputChannel);
    }
```
可以发现，最后是通过调用WMS的addWindow去添加的。具体细节不做深入~~

#### Window的删除过程
删除过程和添加过程类似，我们直接看WindowManagerGlobal的removeView方法，代码如下：

```java
 public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            View curView = mRoots.get(index).getView();
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }

            throw new IllegalStateException("Calling with view " + view
                    + " but the ViewAncestor is attached to " + curView);
        }
    }
```
这边会通过调用findViewLocked方法去拿到待删除View的索引。然后通过removeViewLocked去实现删除逻辑，具体看代码：

```java
 private void removeViewLocked(int index, boolean immediate) {
        ViewRootImpl root = mRoots.get(index);
        View view = root.getView();

        if (view != null) {
            InputMethodManager imm = InputMethodManager.getInstance();
            if (imm != null) {
                imm.windowDismissed(mViews.get(index).getWindowToken());
            }
        }
        boolean deferred = root.die(immediate);
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
                mDyingViews.add(view);
            }
        }
    }
```
这边会调用ViewRootImpl的die方法去处理，我们继续看代码：

```java
 boolean die(boolean immediate) { if (immediate && !mIsInTraversal) {
            doDie();
            return false;
        }
        ···
        mHandler.sendEmptyMessage(MSG_DIE);
        return true;
    }
```
这边会判断同步还是异步，如果是同步就直接调用doDie方法，否则会通过Handler发送消息调用doDie方法去处理，在doDie方法中真正实现删除逻辑的是dispatchDetachedFromWindow方法。主要是做了如下几件事：

- GC回收相关
- 一个IPC过程：mWindowSession.remove(mWindow);
- 调用WindowManagerGlobal.getInstance().doRemoveView(this);去移除关于当前Windows的数据。

#### Window的更新过程
过程基本和添加和删除类似，我们来看代码：

```java
 public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;

        view.setLayoutParams(wparams);

        synchronized (mLock) {
            int index = findViewLocked(view, true);
            ViewRootImpl root = mRoots.get(index);
            mParams.remove(index);
            mParams.add(index, wparams);
            root.setLayoutParams(wparams, false);
        }
    }
```
这段代码主要做了如下几件事：

- 更新view的layoutparms属性
- 找到view的索引
- 移除之前的view
- 添加当前的view
- 设置view的layoutparms属性

### Window的创建过程
我们知道View是依附在Window上的。换句话说有View的地方就有Window。所以Activity、Dialog和Toast都是依附在Window之上的。所以我们来看看这三者的创建过程。

#### Activity的Window创建过程
Activity的Window的创建过程会牵扯到Activity的启动流程。启动流程比较复杂，最后会通过ActivityThread得performLaunchActivity去启动Activity。在此方法中会调用attach方法，为其关联一系列的上下文变量。我们来看看Activity的attch方法：

```java
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
       `···
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;

        mWindow.setColorMode(info.colorMode);
``` 
可以看出Window的具体实现类是PhoneWindow。到这边其实Window就已经创建完成了，Activity会调用setContentView将Window显示出来。

#### Dialog的Window创建过程
Dialog的创建过程基本和Activity类似，具体我们来看Window的构造方法：

```java
  Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
        if (createContextThemeWrapper) {
            if (themeResId == ResourceId.ID_NULL) {
                final TypedValue outValue = new TypedValue();
                context.getTheme().resolveAttribute(R.attr.dialogTheme, outValue, true);
                themeResId = outValue.resourceId;
            }
            mContext = new ContextThemeWrapper(context, themeResId);
        } else {
            mContext = context;
        }

        mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);

        final Window w = new PhoneWindow(mContext);
        mWindow = w;
        w.setCallback(this);
        w.setOnWindowDismissedCallback(this);
        w.setOnWindowSwipeDismissedCallback(() -> {
            if (mCancelable) {
                cancel();
            }
        });
        w.setWindowManager(mWindowManager, null, null);
        w.setGravity(Gravity.CENTER);

        mListenersHandler = new ListenersHandler(this);
    }
```
这边同样是通过PhoneWindow去创建的，创建完成后，依旧是通过setContentView去添加指定布局。我们在来看看show方法：

```java

···
  mDecor = mWindow.getDecorView();

        if (mActionBar == null && mWindow.hasFeature(Window.FEATURE_ACTION_BAR)) {
            final ApplicationInfo info = mContext.getApplicationInfo();
            mWindow.setDefaultIcon(info.icon);
            mWindow.setDefaultLogo(info.logo);
            mActionBar = new WindowDecorActionBar(this);
        }

        WindowManager.LayoutParams l = mWindow.getAttributes();
        if ((l.softInputMode
                & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION) == 0) {
            WindowManager.LayoutParams nl = new WindowManager.LayoutParams();
            nl.copyFrom(l);
            nl.softInputMode |=
                    WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION;
            l = nl;
        }

        mWindowManager.addView(mDecor, l);
        mShowing = true;
···

```
可以看出，在show的时候，它会拿到decorView，然后将DecorView添加到WindowManager中去。


#### Toast的Window创建过程
Toast和Window、Activity的区别就是它具有定时取消的功能，所以其内部有两组IPC过程，一个是INotificationManager，其通过getService拿到一个NotificationManagerService也就是NMS。第二这是NMS的回调也就是TN。具体的实现，我们来看代码：

```java
 public void show() {
        if (mNextView == null) {
            throw new RuntimeException("setView must have been called");
        }

        INotificationManager service = getService();
        String pkg = mContext.getOpPackageName();
        TN tn = mTN;
        tn.mNextView = mNextView;

        try {
            service.enqueueToast(pkg, tn, mDuration);
        } catch (RemoteException e) {
            // Empty
        }
    }
    
    public void cancel() {
        mTN.cancel();
    }
```
这边主要是通过NMS像TN进行一个回调。我们来看看TN的handler处理了什么：

```java
    mHandler = new Handler(looper, null) {
                @Override
                public void handleMessage(Message msg) {
                    switch (msg.what) {
                        case SHOW: {
                            IBinder token = (IBinder) msg.obj;
                            handleShow(token);
                            break;
                        }
                        case HIDE: {
                            handleHide();
                            // Don't do this in handleHide() because it is also invoked by
                            // handleShow()
                            mNextView = null;
                            break;
                        }
                        case CANCEL: {
                            handleHide();
                            // Don't do this in handleHide() because it is also invoked by
                            // handleShow()
                            mNextView = null;
                            try {
                                getService().cancelToast(mPackageName, TN.this);
                            } catch (RemoteException e) {
                            }
                            break;
                        }
                    }
                }
            };
```
我们可以发现其通过handleShow以及handleHide去显示和影藏，我们可以看到在handleShow中他会将Toast的视图去添加到Window中去，具体代码如下：

```java

    mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
    ···
    try {
        mWM.addView(mView, mParams);
        trySendAccessibilityEvent();
        } catch (WindowManager.BadTokenException e) {
        }
```
而在handleHide方法中，它会将当前窗口给移除掉：

```java
    if (mView.getParent() != null) {
        if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
            mWM.removeViewImmediate(mView);
    }
```

### 总结
任何 View 都是附属在一个Window上面的，Window表示一个窗口的概念，也是一个抽象的概念，Window 并不是实际存在的，它是以View的形式存在的。WindowManager 是外界也就是我们访问Window的入口，Window的具体实现位于WindowManagerService 中，WindowManagerService 和 WindowManager 的交互是一个 IPC 过程。

