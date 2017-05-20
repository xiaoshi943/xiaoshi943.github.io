## Handler、Looper源码分析

​	android的消息处理有三个核心类：Looper,Handler和Message。其实还有一个Message Queue（消息队列），但是MQ被封装到Looper里面了，我们不会直接与MQ打交道。

### 一、Looper

​	      该类用于为线程运行一个消息循环，线程默认情况下不带有消息循环。创建一个消息循环的线程，需要在运行消息循环的线程中调用Looper的prepare()方法和loop()方法。

      Looper的字面意思是“循环者”，它被设计用来使一个普通线程变成**Looper**线程。所谓Looper线程就是循环工作的线程。在程序开发中（尤其是GUI开发中），我们经常会需要一个线程不断循环，一旦有新任务则执行，执行完继续等待下一个任务，这就是Looper线程。

      使用Looper类创建Looper线程典型的例子：

```java
class LooperThread extends Thread {
        public Handler mHandler;
  
        public void run() {
            Looper.prepare();
  
            mHandler = new Handler() {
                public void handleMessage(Message msg) {
                    // process incoming messages here
                }
            };
  
            Looper.loop();
        }
    }
```

         可以看到，创建Looper线程最主要的两个方法是prepare()和loop()，我们就从这两个方法入手。

#### 1、prepare()

```java
public final class Looper {
    private static final String TAG = "Looper";
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    private static Looper sMainLooper;  // guarded by Looper.class

    final MessageQueue mQueue;
    final Thread mThread;

    private Printer mLogging;
    public static void prepare() {
        prepare(true);
    }

    // 我们调用该方法会在调用线程的TLS中创建Looper对象
    private static void prepare(boolean quitAllowed) {
	// 试图在有Looper的线程中再次创建Looper将抛出异常
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

```

prepare()背后的工作方式一目了然，其核心就是将looper对象定义为ThreadLocal。这里我们需要对TLS，thread local storage做一下介绍，即线程本地存储。

> 因为存储要么在栈上，例如函数内定义的内部变量。要么在堆上，例如new或者malloc出来的东西但是现在的系统比如Linux和windows都提供了线程本地存储空间，也就是这个存储空间是和线程相关的，一个线程内有一个内部存储空间，这样的话我把线程相关的东西就存储到。（计算机里进程是资源分配和拥有的单位，而线程不拥有资源。按理说无法将一些变量等东西与线程关联，而使用ThreadLocal能解决这一点）。

​	接下来看看Looper(quitAllowed)构造函数。

```java
// 每个Looper对象中有它的消息队列，和它所属的线程
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

#### 2、loop()

​	调用loop方法后，Looper线程就开始真正工作了，它不断从自己的MQ中取出队头的消息(也叫任务)执行。其源码分析如下：

```java
public static void loop() {
        final Looper me = myLooper();//得到当前线程Looper
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue; //得到当前looper的MQ
        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {         
                return;
            }
            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }
			// 非常重要！将真正的处理工作交给message的target，target实际上是				Message中的Handler
           //类型的成员变量
            msg.target.dispatchMessage(msg);
            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }      
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }
            msg.recycle();// 回收message资源
        }
    }
```

       loop()方法中有两个关键的地方，即：**final** Looper me = *myLooper*();和msg.target.dispatchMessage(msg);首先来看*myLooper*()是如何获取当前线程的Looper的。

```java
public static Looper myLooper() {
        return sThreadLocal.get();
    }
```

​	可以看到，myLooper()是通过本地线程*sThreadLocal**获取**Looper**的。为什么能通过**sThreadLocal*获取呢？因为在prepare(boolean quitAllowed)方法中，将Looper对象设置到*sThreadLocal*中了:

```java
private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

**首先，我们知道：** 

1、发送message和接收处理message的是同一个Handler.

2、Handler发送的Message被存放到Looper的MessageQueue中。

3、由Looper不断地从MessageQueue中取出Message进行处理。

**但有几个问题：**

1、Handler与Looper如何关联？

2、Looper的loop()中将Message交个自身的Handler成员变量target进行处理：msg.target.dispatchMessage(msg);  这是否有些奇怪？

 

因此，我们从Handler着手。

### 二、Handler

      Handler扮演了往MQ上添加消息和处理消息的角色（只处理由自己发出的消息），即**通知MQ它要执行一个任务(sendMessage)，并在loop到自己的时候执行该任务(handleMessage)，整个过程是异步的**。handler创建时会关联一个looper，默认的构造方法将关联当前线程的looper，当然还有一个带参的构造方法Handler(Looper looper)或者Handler(Looperlooper, Callback callback)等为Handler指定一个Looper。

      Handler有多个重构的构造方法，我们就看平时用得最多的无参构造方法：

```java
public class Handler {
final MessageQueue mQueue;   // 关联的MQ
    final Looper mLooper; // 关联的looper
    final Callback mCallback;
    final boolean mAsynchronous;
    IMessenger mMessenger;

    public Handler() {
        this(null, false);
    }

    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
	//默认将关联当前线程的looper
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        // 重要！！！直接把关联looper的MQ作为自己的MQ，因此它的消息将发送到关联looper的MQ上
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

      **从上面的代码我们知道Handler和Looper的关联。Handler在初始化的时候，首先获取当前线程的Looper，然后将当前线程Looper的MessageQueue当做自己的MessageQueue，当Handler向自己的MessageQueue发送消息时，其实是向关联的Looper的MessageQuue发送。**

      至于第二个问题：loop()中将Message交个自身的Handler成员变量target进行处理。这个问题我们得看Handler发送消息的机制了，源代码如下。由于发送消息的方法有多个，我们只看发送空消息的方法就足够了。

```java
    public final boolean sendEmptyMessage(int what)
    {
        return sendEmptyMessageDelayed(what, 0);
    }

    public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }

    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this; // message的target必须设为该handler！
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }

```

     **最后我们发现，Handler把自身付给了msg的target。** 

​	**为什么要这么做呢？因为Looper的MessageQueue是一个队列，队列里的Message可能不是都用同一个Handler进行处理，不同的Message可能对应不同的Handler，那如何找到Message对应的Handler呢？这里采用了一种策略，即将Message对应的Handler对象放到自己的成员变量target中，当loop从MessageQueue中取出一个Message后，就将Message交给对应的Handler（即target）进行处理。**



2、接下来我们就看一下Handler是如何处理Message的，因此我们需要了解dispatchMessage()方法。因为在loop()方法中将message交给了dispatchMessage()方法，即：msg.target.dispatchMessage(msg)。

```java
    public void dispatchMessage(Message msg) {
       // 如果message设置了callback，即runnable消息，处理callback！
       if (msg.callback != null) { 
            handleCallback(msg);
        } else {
          // 如果message设置了callback，即runnable消息，处理callback！
            if (mCallback != null) {
           /* 这种方法允许让activity等来实现Handler.Callback接口，避免了自己				编写handler重写handleMessage方法*/
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
           // 如果message没有callback，则调用handler的钩子方法handleMessage
            handleMessage(msg);
        }
    }
```

​	**到这里我们就豁然开朗了，dispatchMessage()方法最后将msg交给handleMessage()方法，handleMessage()方法在Handler中是一个空方法，这就是为什么我们创建一个Handler对象时需要覆写handleMessage()方法，并在该方法中处理消息。** 







t