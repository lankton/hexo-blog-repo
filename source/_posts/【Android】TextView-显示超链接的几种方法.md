---
title: 【Android】TextView 显示超链接的几种方法
date: 2016-07-09 21:47:53
categories: Lan's tech
tags:
  - Android
---
## TextView超链接原理
在这篇博客的开头， 先介绍一下TextView中超链接是如何起作用的。
用户点击文本中的超链接， 会自动生成一个隐式的Intent。这个Intent包含了至少两个信息：action和data。 Action的值为android.intent.action.VIEW， 而data则为超链接的内容。以下文中第一种超链接显示方法为例，点开网址超链接，可以在log中看到这样一条日志：

```
11-15 02:31:01.818 730-1035/? I/ActivityManager: START u0 {act=android.intent.action.VIEW dat=http://www.lankton.cn cmp=com.android.browser/.BrowserActivity (has extras)} from uid 10062 on display 0

```
系统通过该Intent， 选择合适的Activity进行处理， 达到超链接的效果。
## demo展示
文中即将说明的四种超链接方法，分别对应demo中的四行， 请对号入座～
![这里写图片描述](http://img.blog.csdn.net/20151115162013610)
## 显示超链接的几种方式
### 自动显示超链接， 如电话、网络地址等

```java
link_tv = (TextView) this.findViewById(R.id.link_tv);
link_tv.setAutoLinkMask(Linkify.ALL);

String a1 = "hello, 13323332333 www.lankton.cn.";
link_tv.setText(a1);
```
这样电话和网络地址就会在TextView中高亮显示， 并且可以点击跳转。
需要注意的是， 必须先setAutoLinkMask， 再设置文本内容，才会生效。
同样可以在布局文件里设置AutoLink属性达到同样效果。

### 使用html语法
这里演示通过点击超链接， 跳转到另外一个Activity。
首先在Manifest.xml里配置要调转的Activity。

```xml
<activity android:name=".SecondActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:scheme="lankton" />
    </intent-filter>
</activity>
```
这里的重点就是配置了data的scheme， 当点击超链接生成的intent， data的scheme与之相同， 则可以跳转过来。
MainActivity中java代码如下：

```java
link_tv2 = (TextView) this.findViewById(R.id.link_tv2);  link_tv2.setMovementMethod(LinkMovementMethod.getInstance());// 必须加
String a2 = "<a href='lankton://lankton/param1/param2'>second activity</a>";
CharSequence cs = Html.fromHtml(a2);
link_tv2.setText(cs);
```
点击超链接产生如下日志：

```
11-15 02:46:38.616 730-747/? I/ActivityManager: START u0 {act=android.intent.action.VIEW dat=lankton://lankton/param1/param2 cmp=cn.lankton.linkdemo/.SecondActivity (has extras)} from uid 10062 on display 0

```

### 使用SpannableString， 通过ClickableSpan进行设置
这种设置有一个好处， 就是我们可以监听到用户对超链接的点击。现在更常见的是实用其子类URLSpan， 可以传进去一个URL， 这样生成的data就会是这个URL， 而不是超链接内容本身了。

```java
link_tv3 = (TextView) this.findViewById(R.id.link_tv3); 
link_tv3.setMovementMethod(LinkMovementMethod.getInstance());// 必须加
final SpannableString ss1 = new SpannableString("click1, click2, click3");
for (int i = 0; i < 3; i ++) {
    final int cur = i * 8;
    ss1.setSpan(new ClickableSpan() {
        @Override
        public void onClick(View widget) {
            Toast.makeText(MainActivity.this, ss1.subSequence(cur, cur + 6).toString(), Toast.LENGTH_SHORT).show();
        }

        @Override
        public void updateDrawState(TextPaint ds) {
            ds.setColor(Color.RED);
            ds.setUnderlineText(false);
        }
    }, cur, cur + 6, Spanned.SPAN_EXCLUSIVE_EXCLUSIVE);
}
link_tv3.setText(ss1);
```

可以看到， 使用ClickableSpan或其子类， 不仅可以监听点击事件， 还可以自定义超链接样式。这里就把超链接样式改为了红色字体， 无下划线。

### 自定义AutoLink
在开发中，我们会遇到有一些内容，不属于系统默认的超链接格式， 但我们又需要它成为超链接，比如新浪微博的@内容。
这个时候，如果自己逐字符判断， 出现符合的文本字串， 就用ClickableSpan进行设置， 说起来不是不行， 但未免太过费时费力。如果有一种方法， 让系统能像处理默认的超链接那样， 为我们自动选出超链接内容， 岂不美哉。方法只有的， 只需要用一个正则表达式表明你需要的超链接的形式即可。
我之前不知道这个方法， 是从下面这篇博文中学习的。别人辛苦总结的，我就不贴在自己的博客里了。大家有需要就进去学习一下， 会有收获的。
[Android应用实例之---使用Linkify + 正则式区分微博文本链接及跳转处理](http://www.cnblogs.com/ryan1012/archive/2011/07/12/2104087.html)
这个方法生成的链接样式是系统自带的，我对其进行了改良， 更改了颜色， 去掉了下划线。可能会另开一篇博文进行分享。