# android的线程和线程池
### 概述
线程中Android中是很重要的一个概念。从用途上来说，线程分为主线程和子线程。主线程主要用于界面上的交互，而子线程主要用于一些耗时操作。除了thread之外，在Android中可以扮演线程角色的还有很多。例如Asnyctask、IntentService以及handlerthread。对于asnyctask，它的底层用了线程池，对于intentservice和handlerthread来说，它们的底层则是用了线程。

### 主线程和子线程
主线程数指进程所拥有的线程，在Java 中默认情况下，一个进程只有一个主线程。主线程主要用于处理界面的交互，因为用户随时会和界面发生交互，因此主线程主任何时候都需要有较高的响应速度。否则就会产生界面卡顿。这就要求主线程不能执行耗时任务，所以我们就需要子线程。除了工作线程以外的线程都是子线程。子线程也称工作线程，用于执行一些耗时任务，比如网络请求、I/O操作等。

### AsyncTask的工作原理
AsyncTask是一个抽象的范型类，它提供了Params、Progress和Result这三个范型参数，其中Params表示参数的类型，Progress表示后台执行的任务的进度的类型，而Result表示返回结果的类型。如果不需要传具体的参数可以用Void代替。

AsyncTask提供了四个核心方法，具体如下：

- onPreExcute()，在主线程中执行，在异步任务开始前执行，此方法会被调用，一般可以用于一些准备工作。
- doInBackground(Params... params)，在线程池中执行，此方法用于执行异步任务，params表示异步任务输入的参数。在此方法中可以通过publishProgress方法来更新任务进度。publishProgress方法会调用onProgressUpdate方法。另外此方法需要返回结果给onPostExcute方法。
- onProgressUpdate(Progress... values)，在主线程中执行，当后台任务的执行进度发生改变时，此方法会被调用。
- onPostExcute(Result result)，在主线程中执行，在异步任务执行之后，此方法会被调用，其中result时后台任务的返回值。


首先看看asynctask的一些变量到底是干什么的：

```java
    //CPU_COUNT为手机中的CPU核数
    private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    //将手机中的CPU核数加1作为AsyncTask所使用的线程池的核心线程数的大小
    private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
    //将CPU_COUNT * 2 + 1作为AsyncTask所使用的线程池的最大线程数的大小
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    private static final int KEEP_ALIVE = 1;

    //实例化线程工厂ThreadFactory，sThreadFactory用于在后面创建线程池
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        //mCount为AtomicInteger类型，AtomicInteger是一个提供原子操作的Integer类，
        //确保了其getAndIncrement方法是线程安全的
        private final AtomicInteger mCount = new AtomicInteger(1);

        //重写newThread方法的目的是为了将新增线程的名字以"AsyncTask #"标识
        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    //实例化阻塞式队列BlockingQueue，队列中存放Runnable，容量为128
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    //根据上面定义的参数实例化线程池
    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
```
通过以上代码和注释我们可以知道，AsyncTask初始化了一些参数，并用这些参数实例化了一个线程池THREAD_POOL_EXECUTOR，需要注意的是该线程池被定义为public static final，由此我们可以看出AsyncTask内部维护了一个静态的线程池，默认情况下，AsyncTask的实际工作就是通过该THREAD_POOL_EXECUTOR完成的。

我们继续，在执行完上面的代码后，AsyncTask又有如下一条语句：

```java
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
```
上面的代码实例化了一个SerialExecutor类型的实例SERIAL_EXECUTOR，它也是public static final的。SerialExecutor是AsyncTask的一个内部类，代码如下所示：

```java
//SerialExecutor实现了Executor接口中的execute方法，该类用于串行执行任务，
    //即一个接一个地执行任务，而不是并行执行任务
    private static class SerialExecutor implements Executor {
        //mTasks是一个维护Runnable的双端队列，ArrayDeque没有容量限制，其容量可自增长
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        //mActive表示当前正在执行的任务Runnable
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            //execute方法会传入一个Runnable类型的变量r
            //然后我们会实例化一个Runnable类型的匿名内部类以对r进行封装，
            //通过队列的offer方法将封装后的Runnable添加到队尾
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        //执行r的run方法，开始执行任务
                        //此处r的run方法是在线程池中执行的
                        r.run();
                    } finally {
                        //当任务执行完毕的时候，通过调用scheduleNext方法执行下一个Runnable任务
                        scheduleNext();
                    }
                }
            });
            //只有当前没有执行任何任务时，才会立即执行scheduleNext方法
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            //通过mTasks的poll方法进行出队操作，删除并返回队头的Runnable，
            //将返回的Runnable赋值给mActive，
            //如果不为空，那么就让将其作为参数传递给THREAD_POOL_EXECUTOR的execute方法进行执行
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
通过以上代码和注释我们可以知道：

- SerialExecutor实现了Executor接口中的execute方法，该类用于串行执行任务，即一个接一个地执行任务，而不是并行执行任务。
- SerialExecutor内部维护了一个存放Runnable的双端队列mTasks。当执行SerialExecutor的execute方法时，会传入一个Runnable变量r，但是mTasks并不直接存储r，而是又新new了一个匿名Runnable对象，其内部会调用r，这样就对r进行了封装，将该封装后的Runnable对象通过队列的offer方法入队，添加到mTasks的队尾。
- SerialExecutor内部通过mActive存储着当前正在执行的任务Runnable。当执行SerialExecutor的execute方法时，首先会向mTasks的队尾添加进一个Runnable。然后判断如果mActive为null，即当前没有任务Runnable正在运行，那么就会执行scheduleNext()方法。当执行scheduleNext方法的时候，会首先从mTasks中通过poll方法出队，删除并返回队头的Runnable，将返回的Runnable赋值给mActive，如果不为空，那么就让将其作为参数传递给THREAD_POOL_EXECUTOR的execute方法进行执行。由此，我们可以看出SerialExecutor实际上是通过之前定义的线程池THREAD_POOL_EXECUTOR进行实际的处理的。
- 当将mTasks中的Runnable作为参数传递给THREAD_POOL_EXECUTOR执行execute方法时，会在线程池的工作线程中执行匿名内部类Runnable中的try-finally代码段，即先在工作线程中执行r.run()方法去执行任务，无论任务r正常完成还是抛出异常，都会在finally中执行scheduleNext方法，用于执行mTasks中的下一个任务。从而在此处我们可以看出SerialExecutor是一个接一个执行任务，是串行执行任务，而不是并行执行。

除了上述的那些字段，还有如下几个字段：

```java
    //用于通过Handler发布result的Message Code
    private static final int MESSAGE_POST_RESULT = 0x1;
    //用于通过Handler发布progress的Message Code
    private static final int MESSAGE_POST_PROGRESS = 0x2;

    //sDefaultExecutor表示AsyncTask默认使用SERIAL_EXECUTOR作为Executor，
    //即默认情况下AsyncTask是串行执行任务，而不是并行执行任务
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

    //InternalHandler是AsyncTask中定义的一个静态内部类，其绑定了主线程的Looper和消息队列
    private static InternalHandler sHandler;

    //mWorker是一个实现了Callable接口的对象，其实现了Callable接口的call方法
    private final WorkerRunnable<Params, Result> mWorker;
    //根据mFuture是一个FutureTask对象，需要用mWorker作为参数实例化mFuture
    private final FutureTask<Result> mFuture;

    //AsyncTask的初始状态位PENDING
    private volatile Status mStatus = Status.PENDING;

    //mCancelled标识当前任务是否被取消了
    private final AtomicBoolean mCancelled = new AtomicBoolean();
    //mTaskInvoked标识当前任务是否真正开始执行了
    private final AtomicBoolean mTaskInvoked = new AtomicBoolean();
```

我们对以上代码再进行一下说明：

- sDefaultExecutor表示AsyncTask执行任务时默认所使用的线程池，sDefaultExecutor的初始值为SERIAL_EXECUTOR，表示默认情况下AsyncTask是串行执行任务，而不是并行执行任务。

- InternalHandler是AsyncTask中定义的一个静态内部类，其部分源码如下所示：

```java
private static class InternalHandler extends Handler {
    public InternalHandler() {
        //Looper类的getMainLooper方法是个静态方法，该方法返回主线程的Looper
        //此处用主线程的Looper初始化InternalHandler，表示InternalHandler绑定了主线程
        super(Looper.getMainLooper());
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                //发布最后结果
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                //发布阶段性处理结果
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```
通过InternalHandler的构造函数我们可以发现，用主线程的Looper初始化了InternalHandler，说明InternalHandler绑定了主线程。

下面我们看一下asynctask的构造函数：

```java
//AsyncTask的构造函数需要在UI线程上调用
public AsyncTask() {
        //实例化mWorker，实现了Callable接口的call方法
        mWorker = new WorkerRunnable<Params, Result>() {

            public Result call() throws Exception {
                //call方法是在线程池的某个线程中执行的，而不是运行在主线程中
                //call方法开始执行后，就将mTaskInvoked设置为true，表示任务开始执行
                mTaskInvoked.set(true);
                //将执行call方法的线程设置为后台线程级别
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //在线程池的工作线程中执行doInBackground方法，执行实际的任务，并返回结果
                Result result = doInBackground(mParams);
                Binder.flushPendingCommands();
                //将执行完的结果传递给postResult方法
                return postResult(result);
            }
        };

        //用mWorker实例化mFuture
        //在run方法中会调用mWorkder的call方法，具体看FutureTask源码
     mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                //任务执行完毕或取消任务都会执行done方法
                try {
                    //任务正常执行完成
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    //任务出现中断异常
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    //任务执行出现异常
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    //任务取消
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```

- mWorker 
我们之前提到，mWorker其实是一个Callable类型的对象。实例化mWorker，实现了Callable接口的call方法。call方法是在线程池的某个线程中执行的，而不是运行在主线程中。在线程池的工作线程中执行doInBackground方法，执行实际的任务，并返回结果。当doInBackground执行完毕后，将执行完的结果传递给postResult方法。

- mFuture 
mFuture是一个FutureTask类型的对象，用mWorker作为参数实例化了mFuture。在这里，其实现了FutureTask的done方法，我们之前提到，当FutureTask的任务执行完成或任务取消的时候会执行FutureTask的done方法。

在实例化了AsyncTask对象之后，我们就可以调用AsyncTask的execute方法执行任务，代码如下所示：

```java

  @MainThread
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
    
    @MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    //如果当前AsyncTask已经处于运行状态，那么就抛出异常，不再执行新的任务
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    //如果当前AsyncTask已经把之前的任务运行完成，那么也抛出异常，不再执行新的任务
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        //将AsyncTask的状态置为运行状态
        mStatus = Status.RUNNING;

        //在真正执行任务前，先调用onPreExecute方法
        onPreExecute();

        mWorker.mParams = params;
        //Executor的execute方法接收Runnable参数，由于mFuture是FutureTask的实例，
        //且FutureTask同时实现了Callable和Runnable接口，所以此处可以让exec执行mFuture
        exec.execute(mFuture);

        //最后将AsyncTask自身返回
        return this;
    }
```
下面对以上代码进行一下说明：

- 一个AsyncTask实例执行执行一次任务，当第二次执行任务时就会抛出异常。executeOnExecutor方法一开始就检查AsyncTask的状态是不是PENDING，只有PENDING状态才往下执行，如果是其他状态表明现在正在执行另一个已有的任务或者已经执行完成了一个任务，这种情况下都会抛出异常。

- 如果开始是PENDING状态，那么就说明该AsyncTask还没执行过任何任务，代码可以继续执行，然后将状态设置为RUNNING，表示开始执行任务。

- 在真正执行任务前，先调用onPreExecute方法。由于executeOnExecutor方法应该运行在主线程上，所以此处的onPreExecute方法也会运行在主线程上，可以在该方法中做一些UI上的处理操作。

- Executor的execute方法接收Runnable参数，由于mFuture是FutureTask的实例，且FutureTask同时实现了Callable和Runnable接口，所以此处可以让exec通过execute方法在执行mFuture。在执行了exec.execute(mFuture)之后，后面会在exec的工作线程中执行mWorker的call方法，我们之前在介绍mWorker的实例化的时候也介绍了call方法内部的执行过程，会首先在工作线程中执行doInBackground方法，并返回结果，然后将结果传递给postResult方法。

下面我们看一下postResult方法：

```java
private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        //通过getHandler获取InternalHandler，InternalHandler绑定主线程
        //根据InternalHandler创建一个Message Code为MESSAGE_POST_RESULT的Message
        Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        //将该message发送给InternalHandler
        message.sendToTarget();
        return result;
    }
```
在postResult方法中，通过getHandler获取InternalHandle,在得到InternalHandler对象之后，会根据InternalHandler创建一个Message Code为MESSAGE_POST_RESULT的Message，此处还将doInBackground返回的result通过new AsyncTaskResult<Result>(this, result)封装成了AsyncTaskResult，将其作为message的obj属性。构建完成之一，它会发送给InternalHanlder。handler接受如下：

```java
  private static class InternalHandler extends Handler {
        public InternalHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
            AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
            switch (msg.what) {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```
我们再来看看finish的代码：

```java
private void finish(Result result) {
        if (isCancelled()) {
            //如果任务被取消了，那么执行onCancelled方法
            onCancelled(result);
        } else {
            //将结果发传递给onPostExecute方法
            onPostExecute(result);
        }
        //最后将AsyncTask的状态设置为完成状态
        mStatus = Status.FINISHED;
    }
```
finish方法内部会首先判断AsyncTask是否被取消了，如果被取消了执行onCancelled(result)，否则执行onPostExecute(result)方法。需要注意的是InternalHandler是指向主线程的，所以其handleMessage方法是在主线程中执行的，从而此处的finish方法也是在主线程中执行的，进而onPostExecute也是在主线程中执行的。

上面我们知道了任务从开始执行到onPostExecute的过程。我们知道，doInBackground方法是在工作线程中执行比较耗时的操作，这个操作时间可能比较长，而我们的任务有可能分成多个部分，每当我完成其中的一部分任务时，我们可以在doInBackground中多次调用AsyncTask的publishProgress方法，将阶段性数据发布出去。

publishProgress方法代码如下所示：

```java
@WorkerThread
    protected final void publishProgress(Progress... values) {
        if (!isCancelled()) {
            getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                    new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }
```
我们需要将阶段性处理的结果传入publishProgress，其中values是不定长数组，如果任务没有被取消，就会通过InternalHandler创建一个Message对象，该message的Message Code标记为MESSAGE_POST_PROGRESS，并且会根据传入的values创建AsyncTaskResult，将其作为message的obj属性。然后再将该message发送给InternalHandler，然后InternalHandler会执行handleMessage方法，接收并处理该message。

无论任务正常执行完成还是任务取消，都会执行postResultIfNotInvoked方法。postResultIfNotInvoked代码如下所示：

```java
private void postResultIfNotInvoked(Result result) {
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {
            //只有mWorker的call没有被调用才会执行postResult方法
            postResult(result);
        }
    }
```
如果在调用了AsyncTask的execute方法后立马就执行了AsyncTask的cancel方法（实际执行mFuture的cancel方法），那么会执行done方法，且捕获到CancellationException异常，从而执行语句postResultIfNotInvoked(null)，由于此时还没有来得及执行mWorker的call方法，所以mTaskInvoked还未false，这样就可以把null传递给postResult方法。

### HandlerThread
handlerthread继承了thread，它说一种可以使用handler的thread，它的实现也很简单，就是在run方法中通过Looper.prepare()来创建消息队列，并通过
Looper.loop()来开启消息循环，它的run方法如下：

```java
 public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```

### IntentService
IntentService继承于Service。如果我们需要在Service中执行一些耗时操作，就需要用到IntentService。这样我们的工作线程就可以放入到IntentService的onHandleIntent回到方法中。当服务启动时，他会自动执行回掉内的方法。并且，当工作线程执行完成后，他会调用onDestory()方法销毁服务，并不需要我们主动停止服务。(当有多个任务执行的时，它会按照顺序执行完任务，当最后一个任务执行完成之后，它才会停止服务。)

### android中的线程池
使用线程池可以给我们带来很多好处，首先通过线程池中线程的重用，减少创建和销毁线程的性能开销。其次，能控制线程池中的并发数，否则会因为大量的线程争夺CPU资源造成阻塞。最后，线程池能够对线程进行管理，比如使用ScheduledThreadPool来设置延迟N秒后执行任务，并且每隔M秒循环执行一次。

#### ThreadPoolExcutor
Executor作为一个接口，它的具体实现就ThreadPoolExecutor。
Android中的线程池都是直接或间接通过配ThreadPoolExecutor来实现不同特性的线程池。先介绍ThreadPoolExecutor的一个常用的构造方法。

```java

public ThreadPoolExecutor(
//核心线程数，除非allowCoreThreadTimeOut被设置为true，否则它闲着也不会死
int corePoolSize, 
//最大线程数，活动线程数量超过它，后续任务就会排队                   
int maximumPoolSize, 
//超时时长，作用于非核心线程（allowCoreThreadTimeOut被设置为true时也会同时作用于核心线程），闲置超时便被回收           
long keepAliveTime,                          
//枚举类型，设置keepAliveTime的单位，有TimeUnit.MILLISECONDS（ms）、TimeUnit. SECONDS（s）等
TimeUnit unit,
//缓冲任务队列，线程池的execute方法会将Runnable对象存储起来
BlockingQueue<Runnable> workQueue,
//线程工厂接口，只有一个new Thread(Runnable r)方法，可为线程池创建新线程
ThreadFactory threadFactory)

```
ThreadPoolExecutor的各个参数所代表的特性注释中已经写的很清楚了，那么ThreadPoolExecutor执行任务时的心路历程是什么样的呢？（以下用currentSize表示线程池中当前线程数量）

- 当currentSize<corePoolSize时，没什么好说的，直接启动一个核心线程并执行任务。

- 当currentSize>=corePoolSize、并且workQueue未满时，添加进来的任务会被安排到workQueue中等待执行。

- 当workQueue已满，但是currentSize<maximumPoolSize时，会立即开启一个非核心线程来执行任务。

- 当currentSize>=corePoolSize、workQueue已满、并且currentSize>maximumPoolSize时，调用handler默认抛出RejectExecutionExpection异常。

#### 线程池的分类
Android中最常见的四类具有不同特性的线程池分别为FixThreadPool、CachedThreadPool、ScheduleThreadPool以及SingleThreadExecutor。

##### FixThreadPool（一堆人排队上公厕）

```java
public static ExecutorService newFixThreadPool(int nThreads){
    return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
}
//使用
Executors.newFixThreadPool(5).execute(r);
```
- 从配置参数来看，FixThreadPool只有核心线程，并且数量固定的，也不会被回收，所有线程都活动时，因为队列没有限制大小，新任务会等待执行。

- 【前方高能，笔者脑洞】FixThreadPool其实就像一堆人排队上公厕一样，可以无数多人排队，但是厕所位置就那么多，而且没人上时，厕所也不会被拆迁，哈哈o(∩_∩)o ，很形象吧。

- 由于线程不会回收，FixThreadPool会更快地响应外界请求，这也很容易理解，就好像有人突然想上厕所，公厕不是现用现建的。

#####  SingleThreadPool（公厕里只有一个坑位）

```java
public static ExecutorService newSingleThreadPool (int nThreads){
    return new FinalizableDelegatedExecutorService ( new ThreadPoolExecutor (1, 1, 0, TimeUnit. MILLISECONDS, new LinkedBlockingQueue<Runnable>()) );
}
//使用
Executors.newSingleThreadPool ().execute(r);
```
- 从配置参数可以看出，SingleThreadPool只有一个核心线程，确保所有任务都在同一线程中按顺序完成。因此不需要处理线程同步的问题。

- 【前方高能，笔者脑洞】可以把SingleThreadPool简单的理解为FixThreadPool的参数被手动设置为1的情况，即Executors.newFixThreadPool(1).execute(r)。所以SingleThreadPool可以理解为公厕里只有一个坑位，先来先上。为什么只有一个坑位呢，因为这个公厕是收费的，收费的大爷上年纪了，只能管理一个坑位，多了就管不过来了（线程同步问题）。

#####  CachedThreadPool（一堆人去一家很大的咖啡馆喝咖啡）

```java
public static ExecutorService newCachedThreadPool(int nThreads){
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit. SECONDS, new SynchronousQueue<Runnable>());
}
//使用
Executors.newCachedThreadPool().execute(r);
```

- CachedThreadPool只有非核心线程，最大线程数非常大，所有线程都活动时，会为新任务创建新线程，否则利用空闲线程（60s空闲时间，过了就会被回收，所以线程池中有0个线程的可能）处理任务。

- 任务队列SynchronousQueue相当于一个空集合，导致任何任务都会被立即执行。

- 【前方高能，笔者脑洞】CachedThreadPool就像是一堆人去一个很大的咖啡馆喝咖啡，里面服务员也很多，随时去，随时都可以喝到咖啡。但是为了响应国家的“光盘行动”，一个人喝剩下的咖啡会被保留60秒，供新来的客人使用，哈哈哈哈哈，好恶心啊。如果你运气好，没有剩下的咖啡，你会得到一杯新咖啡。但是以前客人剩下的咖啡超过60秒，就变质了，会被服务员回收掉。

- 比较适合执行大量的耗时较少的任务。喝咖啡人挺多的，喝的时间也不长。

##### ScheduledThreadPool（4个里面唯一一个有延迟执行和周期重复执行的线程池）

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize){
return new ScheduledThreadPoolExecutor(corePoolSize);
}
public ScheduledThreadPoolExecutor(int corePoolSize){
super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS, new DelayedQueue ());
}
//使用，延迟1秒执行，每隔2秒执行一次Runnable r
Executors. newScheduledThreadPool (5).scheduleAtFixedRate(r, 1000, 2000, TimeUnit.MILLISECONDS);
```
- 核心线程数固定，非核心线程（闲着没活干会被立即回收）数没有限制。

- 从上面代码也可以看出，ScheduledThreadPool主要用于执行定时任务以及有固定周期的重复任务。


