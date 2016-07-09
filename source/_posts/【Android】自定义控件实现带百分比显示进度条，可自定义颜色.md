---
title: 【Android】自定义控件实现带百分比显示进度条，可自定义颜色
date: 2016-07-09 21:59:41
categories: Lan's tech
tags:
  - Android
---
# 介绍
前天做了一个带百分比显示的条形进度条，效果如下：
![这里写图片描述](http://img.blog.csdn.net/20160110180326105)

# 实现
这个自定义进度条， 看起来简单， 做起来。。。其实也很简单： 主要通过继承View类， 并重写其onDraw方法实现。
思路分为3步：
1. 画进图条背景(图中灰色部分
2. 根据进度画出进度条(图中绿色部分
3. 绘制进度百分比(图中白色文本

前面2个步骤非常简单， 通过drawRoundRect方法进行绘制即可， 第3步也不难， 重点在于定位好绘制文本的位置。文本的水平位置很容易确认， 因为Paint对象提供了measureText方法， 可以获得到文本的长度。用绿色进度条的长度和它做一个减法， 就能得出绘制文本的水平坐标。 
竖直坐标， 就有些复杂了。先看下图（图片来源：http://www.xyczero.com/blog/article/20/）：
![这里写图片描述](http://xyczero.qiniudn.com/Bolg_%E5%A6%82%E4%BD%95%E2%80%9C%E4%BB%BB%E6%80%A7%E2%80%9D%E4%BD%BF%E7%94%A8Android%E7%9A%84drawText%28%29_base.png)

在Canvas对象的drawText方法中， y坐标参数指的是baseline线的y坐标参数。我们所要做的， 就是求出， 当文本垂直居中显示时， 该y坐标的值。
求值， 需要用到Paint的内部类：FontMetrics。

```java
public static class FontMetrics {
        /**
         * The maximum distance above the baseline for the tallest glyph in
         * the font at a given text size.
         */
        public float   top;
        /**
         * The recommended distance above the baseline for singled spaced text.
         */
        public float   ascent;
        /**
         * The recommended distance below the baseline for singled spaced text.
         */
        public float   descent;
        /**
         * The maximum distance below the baseline for the lowest glyph in
         * the font at a given text size.
         */
        public float   bottom;
        /**
         * The recommended additional space to add between lines of text.
         */
        public float   leading;
    }
```
由源码可以看到该类对象提供的几个值的含义。其中：
***ascent*** 代表的就是上图中ascent线的y坐标减去baseline线的y坐标， **所以该值为负数**； 
***descent*** 代表的就是上图中的descent线的y坐标减去baseline的y坐标， **所以该值为正数**；
由此，可知： 文本的高度为2个距离之和， 即2个数字之差：
height ＝ descent - ascent; (1)
又：设空间高度为Height
baseline y坐标 baseY = 1/2 Height + (1/ 2 height - descent); (2)

由(1) (2)式可得：
baseY ＝ 1/2 Height - 1/2 ascent - 1/2 descent;

由此， 需要的数据都被求出来了。
同时， 在values/attrs.xml中添加自定义参数， 使三种颜色可以在布局文件中被配置：

attrs.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>


    <declare-styleable name="RoundedRectProgressBar">
        <attr name="backColor" format="color" />
        <attr name="barColor" format="color" />
        <attr name="textColor" format="color" />
    </declare-styleable>

</resources>
```
自定义进度条RoundedRectProgressBar.java:

```java
package com.landemo.rectprogressbar;

import android.content.Context;
import android.content.res.TypedArray;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.RectF;
import android.util.AttributeSet;
import android.view.View;

/**
 * Created by lankton on 16/1/8.
 */
public class RoundedRectProgressBar extends View {

    private Paint mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    private int barColor;
    private int backColor;
    private int textColor;
    private float radius;

    int progress = 0;

    public RoundedRectProgressBar(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        /*获取自定义参数的颜色值*/
        TypedArray a = context.getTheme().obtainStyledAttributes(attrs, R.styleable.RoundedRectProgressBar, defStyle, 0);
        int n = a.getIndexCount();
        for (int i = 0; i < n; i++)
        {
            int attr = a.getIndex(i);
            switch (attr)
            {
                case R.styleable.RoundedRectProgressBar_backColor:
                    backColor = a.getColor(attr, Color.GRAY);
                    break;
                case R.styleable.RoundedRectProgressBar_barColor:
                    barColor = a.getColor(attr, Color.GREEN);
                    break;
                case R.styleable.RoundedRectProgressBar_textColor:
                                        textColor = a.getColor(attr, Color.WHITE);
                    break;

            }

        }
        a.recycle();
    }

    public RoundedRectProgressBar(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public RoundedRectProgressBar(Context context) {
        this(context, null);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        radius = this.getMeasuredHeight() / 5;
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        //背景
        mPaint.setColor(backColor);
        mPaint.setStyle(Paint.Style.FILL);
        canvas.drawRoundRect(new RectF(0, 0, this.getMeasuredWidth(), this.getMeasuredHeight()), radius, radius, mPaint);
        //进度条
        mPaint.setColor(barColor);
        mPaint.setStyle(Paint.Style.FILL);
        canvas.drawRoundRect(new RectF(0, 0, this.getMeasuredWidth() * progress / 100f, this.getMeasuredHeight()), radius, radius, mPaint);
        //进度
        mPaint.setColor(textColor);
        mPaint.setTextSize(this.getMeasuredHeight() / 1.2f);
        String text = "" + progress + "%";
        float x = this.getMeasuredWidth() * progress / 100 - mPaint.measureText(text) - 10;
        float y = this.getMeasuredHeight() / 2f - mPaint.getFontMetrics().ascent / 2f - mPaint.getFontMetrics().descent / 2f;
        canvas.drawText(text, x, y, mPaint);
    }

    /*设置进度条进度, 外部调用*/
    public void setProgress(int progress) {
        if (progress > 100) {
            this.progress = 100;
        } else if (progress < 0) {
            this.progress = 0;
        } else {
            this.progress = progress;
        }
        postInvalidate();
    }
}

```

然后在MainActivity里添加方法， 调用RoundedRectProgressBar的setProgress方法， 重绘进度条。 这里用Timer对象模拟进度的不断变化。
activity_main.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:paddingBottom="@dimen/activity_vertical_margin"
    android:paddingLeft="@dimen/activity_horizontal_margin"
    android:paddingRight="@dimen/activity_horizontal_margin"
    android:paddingTop="@dimen/activity_vertical_margin"
    android:background="@android:color/white"
    tools:context="com.souche.rectprogressbar.MainActivity">

    <com.souche.rectprogressbar.RoundedRectProgressBar
        android:id="@+id/bar"
        android:layout_width="match_parent"
        android:layout_height="24dp"
        android:layout_marginTop="100dp"
        app:backColor="#E6E6E6"
        app:barColor="#33CC99"
        app:textColor="#FFFFFF"/>

    <Button
        android:id="@+id/btn"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="reset"
        android:layout_centerInParent="true"/>
</RelativeLayout>

```

MainActivity.java

```java
package com.landemo.rectprogressbar;

import android.app.Activity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;

import java.util.Timer;
import java.util.TimerTask;

public class MainActivity extends Activity {

    private RoundedRectProgressBar bar;
    private Button btn;
    private int progress;
    private Timer timer;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        bar = (RoundedRectProgressBar) findViewById(R.id.bar);
        btn = (Button) findViewById(R.id.btn);
        btn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                reset();
            }
        });

    }

    /**
     * 进度条从头到尾跑一次
     */
    private void reset() {
        progress = 0;
        timer = new Timer();
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                bar.setProgress(progress);
                progress ++;
                if (progress > 100) {
                    timer.cancel();
                }
            }
        }, 0, 30);
    }
}

```


