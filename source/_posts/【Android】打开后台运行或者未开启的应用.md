---
title: Android】打开后台运行或者未开启的应用
date: 2016-07-09 21:48:12
categories: Lan's tech
tags:
  - Android
---
思考这个问题的起因是在业务中遇到这样一个场景：应用在后台或者非运行状态下的时候， 点击通知栏的相关通知，发送相应的Broadcast， 相应的receiver需要唤起应用。这里分为2种情况：
1. 应用运行在后台， 则打开应用后， 界面保持为应用最后展示的界面。
2. 应用未开启，则重新启动。 

在网上没有找到合适的解决方案， 自己的解决方案参看以下代码：

```java
    /**
     * 打开应用. 应用在前台不处理,在后台就直接在前台展示当前界面, 未开启则重新启动
     */
    public static void openApplicationFromBackground(Context context) {
        Intent intent;
        ActivityManager am = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
        List<ActivityManager.RunningTaskInfo> list = am.getRunningTasks(100);
        if (!list.isEmpty() && list.get(0).topActivity.getPackageName().equals(context.getPackageName())) {
            //此时应用正在前台, 不作处理
            return;
        }
        for (ActivityManager.RunningTaskInfo info : list) {
            if (info.topActivity.getPackageName().equals(context.getPackageName())) {
                intent = new Intent();
                intent.setComponent(info.topActivity);
                intent.setFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP);
                if (! (context instanceof Activity)) {
                    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                }
                context.startActivity(intent);
                return;
            }
        }
        intent = context.getPackageManager().getLaunchIntentForPackage(context.getPackageName());
        context.startActivity(intent);

    }

```

思路比较简单：
1. 获取当前正在运行的任务栈，取最多100个(应该已经足够):
`List<ActivityManager.RunningTaskInfo> list = am.getRunningTasks(100);`
2. 通过第1个任务栈判断应用是否在前台。如果是， 直接返回不做任何处理；
3. 遍历这100个任务栈的顶层Activity， 判断包名与本应用是否一致。如果一致， 直接通过跳转打开。 同时需要设置FLag      :FLAG_ACTIVITY_SINGLE_TOP  ，避免重复打开topActivity。如果遍历之后没有发现一致， 可以视作应用未打开， 进行第4步操作;
4. 没有在运行任务栈里找到当前应用，直接通过包名启动应用：

```java
intent = context.getPackageManager().getLaunchIntentForPackage(context.getPackageName());
        context.startActivity(intent);
```

经过检验， 达到了目标效果。
