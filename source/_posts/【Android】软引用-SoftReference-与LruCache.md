---
title: 【Android】软引用(SoftReference)与LruCache
date: 2016-07-09 22:04:22
categories: Lan's tech
tags:
  - Android
---
Android开发中， 我们通常需要用到缓存，比如加载图片。使用缓存的好处大家都知道， 比如避免重复访问网络资源、避免重复读取磁盘等， 以提升图片显示速度，这里就不再详述。加载图片使用缓存， 经常会出现OOM(out of memory, 内存不足)。为了避免OOM， 必须要在向内存中加载新资源的同时， 将旧的资源释放。在较早时候， 开发者通常使用软引用解决给问题，而现在， 被广泛使用的方法是使用LruCache。
### 软引用
软引用的原理：当一个内存空间， **只有软引用**指向它时， 当内存不足， GC会将该内存空间回收。
使用方法：
1.创建软引用HashMap作为缓存

```java
private Map<String, SoftReference<Bitmap>> imageCache = new HashMap<String, SoftReference<Bitmap>>();

```

2.向缓存中添加新Bitmap

```java
public void addBitmapToCache(String path) {
        // 强引用的Bitmap对象
        Bitmap bitmap = BitmapFactory.decodeFile(path);
        // 软引用的Bitmap对象
        SoftReference<Bitmap> softBitmap = new SoftReference<Bitmap>(bitmap);
        // 添加该对象到Map中使其缓存
        imageCache.put(path, softBitmap);
    }


```
 注意：由于bitmap为局部变量， 当方法结束时，其指向的内存空间依然只有imageCache中的软引用。
 
3.从缓存中读取Bitmap

```java
public Bitmap getBitmapByPath(String path) {
        // 从缓存中取软引用的Bitmap对象
        SoftReference<Bitmap> softBitmap = imageCache.get(path);
        // 判断是否存在软引用
        if (softBitmap == null) {
            return null;
        }
        // 取出Bitmap对象，如果由于内存不足Bitmap被回收，将取得空
        Bitmap bitmap = softBitmap.get();
        if(bitmap==null){
            return null;
        }
       return bitmap;
    }

```

软引用释放资源是**被动**的， 当内存不足时， GC会对其主动回收。

### LruCache
关于LruCache， 这里就不贴代码了。 因为这个缓存模式不需要开发者自己去实现。这个类包含在android-support-v4包中， 使用方法和其他缓存一样：加载图片前判断缓存中是否已经存在， 如果不存在就重新从图片源加载。
与使用软引用不同， LruCache内部通过一个LinkedHashMap保存资源的强引用。其控制内存的方式是**主动**的，需要在内部记录当前缓存大小， 并与初始化时设置的max值比较，如果超过， 就将排序最靠前(即最近最少使用)的资源从LinkedHashMap中移除。这样， 就没有任何引用指向资源的内存空间了。该内存空间无人认领， 会在GC时得到释放。
关于LinkedHashMap， 其是HashMap的子类， 支持两种排序方式， 第一种是根据插入顺序排序， 第二种就是根据访问进行排序。采用哪种排序方式由其构造函数传入参数决定。在LruCache中， 初始化LinkedHashMap的代码如下：

```java
this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
```
其中最后一个参数， 就是是否根据访问进行排序。

### 异同
二者有很多相似的部分：
1. 无论是通过软引用，还是LruCache， 最后都是通过系统GC到达回收内存的目的；
2. 需要确保没有缓存列表意外的全局强引用指向资源（如果被ImageView显示， 则在ImageView内部也会有对资源的强引用， 要注意）， 否则资源不会得到释放；

也有一些不同的地方， 其中最大的不同就是二者释放内存， 一个是被动的， 一个是主动的（ 虽然结局都是被动地被GC处理）。

### 关于recycle()调用
其实最早在使用LruCache或者软引用的时候， 我产生了这样的疑问：GC可以释放没有强引用指向的内存，但Bitmap的图片资源（像素数据）， 不是保存在native层， 需要显示调用recycle方法进行内存释放吗。而在一些人关于LruCache的博客中， 看到博主回复类似问题，说该操作由LruCache帮助完成了。然而我看遍了LruCache 的源码， 也没有看到哪里有释放底层资源的操作，这反而更加深了我的疑惑。
后来在网上看到了这样的说明， 即在Android 3.0(Level 11)及其以后， Bitmap的像素数据与Bitmap的对象一起保存在java堆中， 如此， 系统GC时， 也可以一起将像素资源回收了。
**要注意的是**， 在使用LruCache时， 千万不要画蛇添足， 在LruCache的entryRemoved回调中实现对释放资源的手动recycle。 因为虽然该Bitmap从LinkedHashMap中被移除了， 但我们无法得知外部是否还有对当前Bitmap的引用。如果还有ImageView正显示着该图片， 那必然会导致崩溃。

### 最适用场景
从以上总结可以看出，两种方案依靠的都是系统GC，所以当有外部强引用， 包括有ImageView正在使用Bitmap时， 这种方案可以说并无卵用。但是在ListView以及GridView这样的场景下， 由于item的视图资源不断回收再利用，就的ImageView会使用新的Bitmap， 则会失去对就Bitmap的强引用。旧的Bitmap就可以放心去了。。。

### 取舍
弱引用实现缓存， 有一个必要条件就是它在系统内存不足时才会被释放，而从 Android 2.3 (API Level 9)开始，垃圾回收器会更倾向于回收持有软引用或弱引用的对象，这让软引用和弱引用变得不再可靠， 即在内存充足的情况下， 它们指向的对象依然有可能被回收。如此， 软引用Map做缓存， **缓存命中率会变低**，效果就会大打折扣。
所以， 使用“主动“方式， 来实现对内存控制的LruCache成为了现在实现内存缓存的主流方式， 显然更可靠，也更值得推荐。
