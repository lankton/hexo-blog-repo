---
title: 【Android】自定义控件实现可滑动的开关(switch)
date: 2016-07-09 21:15:48
categories: Lan's tech
tags:
  - Android
---
# 介绍
昨天晚上写了一个Android的滑动开关， 即SlideSwitch。效果如下：
![这里写图片描述](http://img.blog.csdn.net/20150701024352476)
# 实现
实现的思路其实很简单，监听控件上的touch事件，并不断刷新，让滑块在手指的位置上绘出，达到滑块跟着手指滑动的显示效果。
先看一下代码：
SlideSwitch.java (7月3日有修改：在touch事件里调用onStateChangedListener前增加判空)

```java
package com.incell.view;

import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.View;

public class SlideSwitch extends View{
    
    private Paint mPaint = new Paint(Paint.ANTI_ALIAS_FLAG); //抗锯齿
    
    boolean isOn = false;
    float curX = 0;
    float centerY; //y固定
    float viewWidth;
    float radius;
    float lineStart; //直线段开始的位置（横坐标，即
    float lineEnd; //直线段结束的位置（纵坐标
    float lineWidth;
    final int SCALE = 4; // 控件长度为滑动的圆的半径的倍数
    OnStateChangedListener onStateChangedListener;

    public SlideSwitch(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
    }
 
    public SlideSwitch(Context context, AttributeSet attrs) {
        super(context, attrs);
    }
    
    public SlideSwitch(Context context) {
        super(context);
    }
    
    
    
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        // TODO Auto-generated method stub
        curX = event.getX();
        if(event.getAction() == MotionEvent.ACTION_UP)
        {
            if(curX > viewWidth / 2)
            {
                curX = lineEnd;
                if(false == isOn)
                {
                    //只有状态发生改变才调用回调函数， 下同
                    if(null != onStateChangedListener)
                    {
                        onStateChangedListener.onStateChanged(true);
                    }
                    isOn = true;
                }
            }
            else
            {
                curX = lineStart;
                if(true == isOn)
                {
                    if(null != onStateChangedListener)
                    {
                        onStateChangedListener.onStateChanged(false);
                    }
                    isOn = false;
                }
            }
        }
        /*通过刷新调用onDraw*/
        this.postInvalidate();
        return true;
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // TODO Auto-generated method stub
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        /*保持宽是高的SCALE / 2倍， 即圆的直径*/
        this.setMeasuredDimension(this.getMeasuredWidth(), this.getMeasuredWidth() * 2 / SCALE);
        viewWidth = this.getMeasuredWidth();
        radius = viewWidth / SCALE;
        lineWidth = radius * 2f; //直线宽度等于滑块直径
        curX = radius;
        centerY = this.getMeasuredWidth() / SCALE; //centerY为高度的一半
        lineStart = radius;
        lineEnd = (SCALE - 1) * radius;
    }

    @Override
    protected void onDraw(Canvas canvas) {
        // TODO Auto-generated method stub
        super.onDraw(canvas);
        
        /*限制滑动范围*/
        curX = curX > lineEnd?lineEnd:curX;
        curX = curX < lineStart?lineStart:curX;
        
        /*划线*/
        mPaint.setStyle(Paint.Style.STROKE);
        mPaint.setStrokeWidth(lineWidth);
        /*左边部分的线，绿色*/
        mPaint.setColor(Color.BLUE);
        canvas.drawLine(lineStart, centerY, curX, centerY, mPaint);
        /*右边部分的线，灰色*/
        mPaint.setColor(Color.GRAY);
        canvas.drawLine(curX, centerY, lineEnd, centerY, mPaint);
        
        /*画圆*/
        /*画最左和最右的圆，直径为直线段宽度， 即在直线段两边分别再加上一个半圆*/
        mPaint.setStyle(Paint.Style.FILL);
        mPaint.setColor(Color.GRAY);
        canvas.drawCircle(lineEnd, centerY, lineWidth / 2, mPaint);  
        mPaint.setColor(Color.BLUE);
        canvas.drawCircle(lineStart, centerY, lineWidth / 2, mPaint);
        /*圆形滑块*/
        mPaint.setColor(Color.LTGRAY);
        canvas.drawCircle(curX, centerY, radius , mPaint);
        
    }
    /*设置开关状态改变监听器*/
    public void setOnStateChangedListener(OnStateChangedListener o)
    {
        this.onStateChangedListener = o;
    }
    
    /*内部接口，开关状态改变监听器*/
    public interface OnStateChangedListener
    {
        public void onStateChanged(boolean state);
    }

}

```

注释应该很详细了。主要有以下几点。
1、重写了onMeasure方法，**使控件高度依赖于控件的宽度**。这样不论在布局文件中如何设置，总能**保证控件的宽高比**。
2、控制好滑块的活动范围
3、定义内部接口OnStateChangedListener，并在自定义控件里定义了其对象以及从外部赋值的方法setOnStateChangedListener，以便**对开关状态更改事件进行监听并调用回调**。

# 使用及Demo
在布局文件中添加该控件即可使用。Demo效果为动图展示效果（demo里颜色为绿色，动图为蓝色是因为绿色会导致截取gif时出问题，临时更改的）。
Demo中布局文件如下：
activity_main.xml:

```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <com.example.slideswitchexample.SlideSwitch
        android:id="@+id/slide_switch"
        android:layout_width="200dp"
        android:layout_height="wrap_content"
        android:layout_centerInParent="true"/>

</RelativeLayout>

```
Demo中Activity代码如下：
MainActivity.java
```java
package com.example.slideswitchexample;

import com.example.slideswitchexample.SlideSwitch.OnStateChangedListener;

import android.app.Activity;
import android.os.Bundle;
import android.widget.Toast;

public class MainActivity extends Activity {

    SlideSwitch sSwitch;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        sSwitch = (SlideSwitch) this.findViewById(R.id.slide_switch);
        sSwitch.setOnStateChangedListener(new OnStateChangedListener(){

            @Override
            public void onStateChanged(boolean state) {
                // TODO Auto-generated method stub
                if(true == state)
                {
                    Toast.makeText(MainActivity.this, "开关已打开", 1000).show();
                }
                else
                {
                    Toast.makeText(MainActivity.this, "开关已关闭", 1000).show();
                }
            }
            
        });
    }


}

```

[点此下载Demo工程](http://download.csdn.net/detail/u013015161/8856597)