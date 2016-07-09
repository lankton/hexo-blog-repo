---
title: 【Android】利用Activity生命周期监听应用前后台切换
date: 2016-07-09 21:37:51
categories: Lan's tech
tags:
  - Android
---
# 实现介绍
在Android应用开发中，我们有时候需要监听到应用前后台的切换。这里提供一种思路，**该思路并非原创**，而是一种比较通用的办法，这里做一下介绍，附带实际过程中遇到的问题的解决。
具体实现思路是通过重写Activity的onResume方法和onStop方法实现，即在onStop里判断应用是否切换到后台，在onResume里判断是否切换到前台。
先回顾一下Activity生命周期：
![这里写图片描述](http://hi.csdn.net/attachment/201109/1/0_1314838777He6C.gif)
当Activity完全不可见时，执行onStop。这个时候我们判断应用是否还在前台，这样就监听到了***前台切后台***，可以做相关处理，同时置全局标志位。

```java
@Override
    protected void onStop() {
        // TODO Auto-generated method stub
        super.onStop();
        if(!Utils.isForeground(this))
        {
            Utils.isActive = false;
        }
    }
```
判断应用是否前台的方法isForeground如下

```java
/*判断应用是否在前台*/
    public static boolean isForeground(Context context)
    {
        ActivityManager am = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
        List<RunningTaskInfo> tasks = am.getRunningTasks(1);
        if (!tasks.isEmpty()) {
            ComponentName topActivity = tasks.get(0).topActivity;
            if (topActivity.getPackageName().equals(context.getPackageName())) {
                return true;
            }
        }
        return false;
    }
```
当Activity恢复为可见时，执行onResume。这是就可以通过全局标志位isActive来判断场景。如果isActive为false，则Activity的出现时因为应用从后台切换到前台，这样就监听到了***后台切前台***，可以做相关处理，同时要记得恢复标志位isActive为true。如果为true，则代表仅仅是应用内部的视图切换。

```java
@Override
    protected void onResume() {
        // TODO Auto-generated method stub
        super.onResume();
        if(!Utils.isActive)
        {
            Utils.isActive = true;
            /*一些处理，如弹出密码输入界面*/
        }
    }
```
由于是监听整个应用的前后台切换，所以上述重写可以实现在一个继承了Activity的父类***BaseActivity***里，直接继承Activity的类改为继承BaseActivity。如果有部分Activity是继承了FragmentActivity等而非直接继承Activity，同样建议新建一个父类，如BaseFragmentActivity等，在其中重写onResume，onStop，然后被继承。

# 遇到问题
在使用上述方法中，已经可以很好的实现监听应用前后台切换。而遇到的这个问题恰恰是由于在某个场景下，我不希望这种前后台切换被监听到。
上述代码，从注释里看到，onResume里检测到后台切前台，会让用户重新输入密码来保证安全，这样就出现了一个问题：
我在应用里调用了系统相机进行拍照，拍照完成之后应该回到应用进行图片展示，却直接弹出输入密码的界面。于是我每次拍完照都要手动输一次密码。
这种情况描述起来就是，虽然我调用了系统应用来拓展本应用的功能，但却不应该让用户感觉出他们曾经离开了这个应用。（可能描述比较混乱，但大概就是这个意思。。。）
解决办法：
Activity生命周期里还应该有一个回调方法，但这个方法在描述Activity的生命周期的时候经常被忽视掉：***onActivityResult***。通过试验，发现***onActivityResult在onResume之前执行***。所以解决办法如下：
通过startActivityForResult调用系统相机，传入requestCode。然后在onActivityResult里，判断requestCode。如果是从请求相机返回，则提前对前后台标志位isActive置true，这样onResume里就不会监听到后台切前台事件了。

```java
 @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        // TODO Auto-generated method stub
        super.onActivityResult(requestCode, resultCode, data);
        if(Utils.REQUEST_CEMERA == requestCode) {
            /*处理数据等*/
            ......
            Utils.isActive = true;
        }
    }
```
这篇博文重点还是前面介绍如何监听前后台切换，后面解决的问题没有普遍性，可能对大家没什么参考价值，就当自己做一个记录了。