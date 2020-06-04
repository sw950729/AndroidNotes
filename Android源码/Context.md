# Context
### Context基本概念
先看一下Context源码，代码如下：  

```java
/**
 * Interface to global information about an application environment.  This is
 * an abstract class whose implementation is provided by
 * the Android system.  It
 * allows access to application-specific resources and classes, as well as
 * up-calls for application-level operations such as launching activities,
 * broadcasting and receiving intents, etc.
 */
public abstract class Context {
    ...
}    
```
解释如下： 
- 有关应用程序环境的全局信息的接口。这是一个抽象类，其实现由Android系统提供。, 它允许访问特定于应用程序的资源和类，以及用于应用程序级操作的上调，activity，广播和intent等。

下面看看Context的实现类ContextImpl源码注释：

```java

/**
 * Common implementation of Context API, which provides the base
 * context object for Activity and other application components.
 */
class ContextImpl extends Context {
    ...
}
```
解释如下：
- Context API的通用实现，它为Activity和其他应用程序组件提供基础上下文对象。

再来看看Context的包装类ContextWrapper源码注释：

```java
/**
 * Proxying implementation of Context that simply delegates all of its calls to
 * another Context.  Can be subclassed to modify behavior without changing
 * the original Context.
 */
public class ContextWrapper extends Context {
    Context mBase;
    ...
}
```
解释如下：
- 代理Context的实现，简单地将其所有调用委托给另一个Context。可以进行子类化以修改行为而不更改原始上下文。 

该类包括了一个真正的Context对象，也就是ContextImpl。其是ContextImpl的代理模式。

### 一个应用有多少个Context
面试最常问的就是一个应用有多少个Context，有多少个Window。  
Android应用程序只有四大组件，通过源码，我们看出，只有Activity和Service是继承了Context的。当然，每个应用还有额外的一个Application。所以总数为：

```
APP Context总数 = Application数(1) + Activity数(number) + Service数(number);
```

### Context在ActivityThread的实例化
首先，肯定有人问为什么context是在ActivityThread中实例化的，这边不具体深入启动流程。有兴趣的可以自己去看。

#### Activity中ContextImpl的实例化
通过startActivity我们会启动一个新的Activity。它最终会调用ActivityThread的performLaunchActivity方法，具体如下：

```java
  private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ...
        ContextImpl appContext = createBaseContextForActivity(r);
    
        ...
        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (activity != null) {
                ...
                appContext.setOuterContext(activity);
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);
                ...
                
            }
            ...
        }
        ...
     return activity;
  }
```
上述代码差不多就是通过createBaseContextForActivity(r);创建appContext，然后通过activity.attach设置。现在我们看看createBaseContextForActivity的源码：

```java
   private Context createBaseContextForActivity(ActivityClientRecord r,
            final Activity activity) {
       ...  
        ContextImpl appContext = ContextImpl.createActivityContext(this, r.packageInfo, r.token);
        appContext.setOuterContext(activity);
        Context baseContext = appContext;
        ......
        return baseContext;
    }
```
可以看到是通过ContextImpl去创建一个activity的上下文，然后通过setOuterContext去将当前的activty和context进行绑定。

在来看看attach方法：

```java
  final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        attachBaseContext(context);
        ...
    }
```

通过上述代码，我们可以看出通过ContextThemeWrapper类的attachBaseContext方法，将createBaseContextForActivity中实例化的ContextImpl对象传入到ContextWrapper类的mBase变量，这样ContextWrapper（Context子类）类的成员mBase就被实例化为Context的实现类ContextImpl。

#### Service中的ContextImpl实例化
通过startService或者bindService我们会启动一个新的Service。它最终会调用ActivityThread的handleCreateService方法，具体如下：

```java
 private void handleCreateService(CreateServiceData data) {
        ......
        
        Service service = null;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            ......
        }

        try {
            ......
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);

            Application app = packageInfo.makeApplication(false, mInstrumentation);
            service.attach(context, this, data.info.name, data.token, app,ActivityManagerNative.getDefault());
            service.onCreate();
            ......
        } catch (Exception e) {
            ......
        }
    }
```
具体和上面的activity类型，这边不深入，我们看看service的attach方法：

```java
  public final void attach(
            Context context,
            ActivityThread thread, String className, IBinder token,
            Application application, Object activityManager) {
        attachBaseContext(context);
        .....
    }
```
可以看出步骤流程和Activity的类似，只是实现细节略有不同而已。

#### Application中的Context实例化
前面我们说到一个应用有一个Application，它的生命周期伴随着整个项目。创建Application的过程也是在ActivityThread中的handleBindApplication方法中操作。Application的创建是在该方法中调运LoadedApk类的makeApplication方法中实现，LoadedApk类的makeApplication()方法中源代码如下：

```java
public Application makeApplication(boolean forceDefaultAppClass, Instrumentation instrumentation) {  
    ...  
    try {  
        java.lang.ClassLoader cl = getClassLoader();
        ContextImpl appContext = new ContextImpl();    
        appContext.init(this, null, mActivityThread);  
        app = mActivityThread.mInstrumentation.newApplication(  
                cl, appClass, appContext);                   
        appContext.setOuterContext(app); 
    }   
    ...  
}  
```
可以看出它将appContext通过Instrumentation的newApplication传进去了，下面我们看看Instrumentation的newApplication方法：

```java
  public Application newApplication(ClassLoader cl, String className, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        return newApplication(cl.loadClass(className), context);
    }
    
    static public Application newApplication(Class<?> clazz, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        ......
        app.attach(context);
        return app;
    }
```
我们接着看Application的attach方法：

```java
final void attach(Context context) {
        attachBaseContext(context);
        ......
    }
```
可以看出，和activity与service类似，只是实现的细节变了。

### getApplication和getApplicationContext的区别
很多人分不清getApplication和getApplicationContext，这其中包括我，于是就去翻了翻源码：

```java
public class Activity extends ContextThemeWrapper
        implements LayoutInflater.Factory2,
        Window.Callback, KeyEvent.Callback,
        OnCreateContextMenuListener, ComponentCallbacks2,
        Window.OnWindowDismissedCallback {
    ......
    public final Application getApplication() {
        return mApplication;
    }
    ......
}

class ContextImpl extends Context {
    ......
    @Override
    public Context getApplicationContext() {
        return (mPackageInfo != null) ?
                mPackageInfo.getApplication() : mMainThread.getApplication();
    }
    ......
}
```
可以看到getApplicationContext方法是Context的方法，而且返回值是Context类型，返回对象和上面通过Service或者Activity的getApplication返回的是一个对象。

所以说对于客户化的第三方应用来说两个方法返回值一样，只是返回值类型不同，还有就是依附的对象不同而已。

### 注意事项
从上面我们分析了context，但我们要合理的利用的context对象，context与其所关联的对象生命周期相同，所以我们需要避免出现没有必要的内存泄漏。

