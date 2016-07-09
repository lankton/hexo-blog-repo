---
title: 【Android】在不同的线程池中执行AsyncTask
date: 2016-07-09 21:38:21
categories: Lan's tech
tags:
  - Android
---
说起AsyncTask，有过Android开发经历的人应该都很熟悉，这是我们异步执行耗时操作的一个利器。
在一般情况下，如果有若干通过execute()方法执行的AsyncTask对象，这些的对象的异步操作会按顺序一个一个执行。这是因为使用execute方法的AsyncTask，会默认调用一个static的线程池变量THREAD_POOL_EXECUTOR进行管理。该线程池保证了各AsyncTask执行时的时序，即一次执行一个，先进先出。
(如果想要更加深入的了解AsyncTask的工作原理，推荐博文：[Android应用AsyncTask处理机制详解及源码分析](http://blog.csdn.net/yanbober/article/details/46117397)

**在这种情况下，可能会遇到以下问题**：
比如类似新浪微博的应用， 进入首页时启动了N个AsyncTask，来从网络加载首页所需要的图片。在首页的图片没有全部加载成功之前，点进某一条微博查看详细内容，同样使用了AsyncTask加载图片。这个时候我们可能会等待非常久的时间才能看到图片，因为需要等首页的那N个AsyncTask执行完，才会执行详细页面的AsyncTask。这样就可能造成很不好的体验。这种情况下，我们可能需要让首页和详细页面的AsyncTask在不同的线程池中执行，以避免这种在同一个默认线程池中执行导致出现的等待现象。

***如果要在非默认的线程池(THREAD_POOL_EXECUTOR)中执行AsyncTask***，则需要使用***excuteOnExecutor***方法代替execute, 因为该方法中允许我们传入自己定义的线程池。

Java通过Executors类的四个方法，分别提供了4种不同的线程池。
***newCachedThreadPool***  创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。
***newFixedThreadPool***  创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。
***newScheduledThreadPool***  创建一个定长线程池，支持定时及周期性任务执行。
***newSingleThreadExecutor***  创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。
（以上对四种方法的简介摘自Trinea: [介绍new Thread的弊端及Java四种线程池的使用,对Android同样适用](http://www.trinea.cn/android/java-android-thread-pool/)）

如果想让AsyncTask在不同的线程中执行，同时一个线程池中的AsyncTask依然一个一个执行，则可以选用singleThreadExecutor。
示例代码：

```java
ExecutorService loadExecutor = Executors.newSingleThreadExecutor();
LoadSmallPicTask task = new LoadSmallPicTask(photoUrl);
task.executeOnExecutor(loadExecutor);

```

在其他几种线程池中执行AsyncTask的方法跟上面相同。
