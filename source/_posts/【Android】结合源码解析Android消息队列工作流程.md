---
title: 【Android】结合源码解析Android消息队列工作流程
date: 2016-07-09 22:11:29
categories: Lan's tech
tags:
  - Android
---
## 前言
最近在对一些Android比较基础的知识做一些回顾。回顾到消息队列部分， 便想着结合源码做一篇关于Android消息队列的讲解。然而，我深知这个主题已经被各种翻来覆去地讲， 各种刨根挖底地讲， 各种XXXX地讲... ...各位同学应该也已经看烦了。 但是我还是决定写这么一篇博客。。。  

在整个博文行进过程中， 我会用自己的思路进行组织， 先细讲Looper循环， 再细讲Handler对消息的处理，对源码的截选也做了排布， 希望比较其它类似的文章， 能有更加清楚、明了的叙述。(just hope

## 简例
在博客的开头， 我们先看一段代码， 即消息队列的常见使用方式。  

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new Thread(new Runnable() {
            @Override
            public void run() {
                Looper.prepare();
                final Handler handler = new Handler() {
                    @Override
                    public void handleMessage(Message msg) {
                        super.handleMessage(msg);
                        Log.v("handlerdemo", "" + msg.what);
                    }
                };
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        Message msg = Message.obtain();
                        msg.what = 9;
                        handler.sendMessage(msg);
                    }
                }).start();
                Looper.loop();
            }
        }).start();
    }
}
```
这个demo很简单， 在一个新线程a中， 调起另一个新线程b， 并用handler从b中发送消息， 在a线程中打印出相关信息。（为了保证demo的一般性， a线程也用的新线程， 而不是UI线程）  
## 消息队列的循环
### 1. 在线程入口方法开头调用Loop.prepare

```java
public static void prepare() {
        prepare(true);
}
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```
可以看到， 调用该方法， 创建了一个Looper对象， 并将其保存在ThreadLocal对象中。sThreadLocal是Looper类的static对象，在不同线程中， 各自set进去的对象相互独立。

创建Looper对象的代码如下：

```
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```
创建Looper的过程中， 也创建了消息队列(MessageQueue)，并作为final 属性保存在Looper对象中。
即：  
**一个Looper， 对应一个MessageQueue**  
### 2.在线程入口方法结尾执行Looper.loop方法###

```java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;

    // Make sure the identity of this thread is that of the local process,
    // and keep track of what that identity token actually is.
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        // This must be in a local variable, in case a UI event sets the logger
        Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }

        msg.target.dispatchMessage(msg);

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        // Make sure that during the course of dispatching the
        // identity of the thread wasn't corrupted.
        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
            Log.wtf(TAG, "Thread identity changed from 0x"
                    + Long.toHexString(ident) + " to 0x"
                    + Long.toHexString(newIdent) + " while dispatching to "
                    + msg.target.getClass().getName() + " "
                    + msg.callback + " what=" + msg.what);
        }

        msg.recycleUnchecked();
    }
}
```
代码很长， 我们重点看其中跟消息机制流程有关的部分.可以看到， loop的过程通过一个
```java
for (;;)
```
来进行死循环, 不断从MessageQueue中取出消息， 分发给Message对应的target(即Handler）进行处理：
```java
msg.target.dispatchMessage(msg);
```
而这个消息队列， 则是从与当前Thread相关联的Looper中取出:

```java
final Looper me = myLooper();
if (me == null) {
    throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
}
final MessageQueue queue = me.mQueue;
```
这里可以看一下Looper.myLooper方法了。还记得前面说的ThreadLocal对象吗， 那时set了一个looper进去， 在这里就把取出来了:  

```java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```
现在， 消息的循环机制已经很清楚了， 接下来要看的就是消息是如何被添加进队列， 以及如何被处理的了。  
## Handler发送以及处理消息
Handler发送消息， 主要有两种形式， 分别是send方式和post方式，两者各有sendMessageDelayed、 postDelayed等多重形式。而post方式， 最终调用的也是send方式。
eg：

```java
public final boolean postDelayed(Runnable r, long delayMillis)
{
    return sendMessageDelayed(getPostMessage(r), delayMillis);
}
private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}
```
**虽然post方式， 传进的参数是Runnable对象， 但依然会通过该对象组装一个Message对象， 传给相应的send方法。**于是我们只看send方法就好了～  

```java
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
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
可以看到， **Handler发送消息的过程， 其实就是将消息放进MessageQueue队列(mQueue)的过程，同时将Message对象的target属性指向自己。**
有人可能好奇， 在Looper对象中， 我们不是已经有了一个消息队列mQueue了吗。 我们进入Handler的构造器，来看看Handler里的mQueue是什么。  

```java
 public Handler(Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }

    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
    }
```
myLooper方法， 上面已经介绍过了。那么， mQueue是什么就很清楚了：**就是当前Thread所对应Looper的mQueue。我们的消息队列，一直就只有这一个。**

Handler如何处理从Looper循环从消息队列里取出的消息呢。上面我们已经在loop方法里看到了处理的入口:

```java
msg.target.dispatchMessage(msg);
```
target是什么？ 上面已经说了， 就是把消息塞进队列的Handler。看一下Handler中的相关方法:

```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
private static void handleCallback(Message message) {
    message.callback.run();
}
```
代码非常简单。首先判断Message对象的callback属性， 即上面post方式中作为参数的Runnable对象， 不为空则直接调用其中run方法。
如果callback为空，则判断Handler对象中的mCallback属性。 这个属性不是Runnable对象， 而是Handler.Callback对象。在上述Handler构造器代码中，可以看到其被形参赋值，。

```java
public interface Callback {
    public boolean handleMessage(Message msg);
}
```
如果这两个分支全部走空，才轮到Hanlder对象的handleMessage方法， 对Message对象进行处理。
这就是Handler对Message对象的发送和处理流程。  
## 在主线程中使用Looper？
有朋友可能会有疑问， 在主线程中我们并没有使用Looper.prepare方法为线程产生Looper对象， 也没有使用Looper.loop方法对消息队列进行循环取数据， 为什么还能使用Handler。答案很简单， 那就是Android已经帮我们做了。  
主线程不是由我们调起， 其入口也不是我们定义。在其入口main方法的开头， 调用了Looper.prepareMainLooper()

```java
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}  
```  
可以看到， 调用了普通的prepare方法。  
在main方法的结尾， 调用了Looper.loop方法。  
所以在主线程中， 我们放心地使用Handler就好了。

## 结言
**谢谢看到这里！**
