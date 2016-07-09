---
title: 【Android】自定义控件实现九宫格解锁
date: 2016-07-09 21:15:06
categories: Lan's tech
tags:
  - Android
---
## 介绍
这两天写了一个九宫格锁屏的控件，实现了九宫格锁屏的设置和解锁。该控件没有使用任何图片资源，显示的内容（包括点、圆、线等）全部由画笔绘制，所以可以自由复用。
使用效果图：
![这里写图片描述](http://img.blog.csdn.net/20150629234556437)

## 实现
先上代码吧。
自定义九宫格控件：LocusPassView

```java
package com.example.locusexample;

import java.util.ArrayList;
import java.util.List;
import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.PointF;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.View;

public class LocusPassView extends View{
    
    private Paint mPaint = new Paint(Paint.ANTI_ALIAS_FLAG); //抗锯齿
    private PointF[][] mPoints = new PointF[3][3];
    private List<Integer> pathNodes = new ArrayList<Integer>();
    private float centerRadius; //每个实心点的半径
    private float circleRadius; //空心圆半径
    private float viewWidth;
    private float viewHeight;
    private float curX = 0;
    private float curY = 0;
    
    
    private OnCompleteListener onCompleteListener = null;
    
    public LocusPassView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
    }
    public LocusPassView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }
    public LocusPassView(Context context) {
        super(context);
    }
    
    
    
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        // TODO Auto-generated method stub
        if(event.getAction() == MotionEvent.ACTION_DOWN 
                || event.getAction() == MotionEvent.ACTION_MOVE )
        {
            curX = event.getX();
            curY = event.getY();
            detectGetPoint(curX, curY);
        }
        else if(event.getAction() == MotionEvent.ACTION_UP)
        {
            if(pathNodes.size() >= 3)
            {
                if(null != onCompleteListener)
                {
                    onCompleteListener.onComplete(pathToString(pathNodes));
                }
            }
            pathNodes.clear();
        }
        this.postInvalidate();
        return true;
    }
    @Override
    protected void onDraw(Canvas canvas) {
        // TODO Auto-generated method stub
        super.onDraw(canvas);
        viewWidth = this.getMeasuredWidth();
        viewHeight = this.getMeasuredHeight();
        centerRadius = viewWidth / 24;
        circleRadius = viewWidth / 6 * 3 / 5;
        drawPoints(canvas);
        drawLines(canvas, curX, curY);
    }
    public void drawPoints(Canvas canvas)
    {
        mPaint.setColor(Color.BLUE);
        mPaint.setStyle(Paint.Style.FILL);
        for(int i = 0; i < 3; i++)
        {
            for(int j = 0; j < 3; j++)
            {
                mPoints[i][j] = new PointF((int)(viewWidth / 6 + viewWidth / 3 * j),
                        (int)(viewHeight / 6 + viewHeight / 3 * i));
                canvas.drawCircle(mPoints[i][j].x, mPoints[i][j].y, centerRadius, mPaint);
                
            }
        }
        mPaint.setStyle(Paint.Style.STROKE);
        mPaint.setStrokeWidth(centerRadius / 6);
        for(int i = 0; i < pathNodes.size(); i++)
        {
            int m = pathNodes.get(i) / 3;
            int n = pathNodes.get(i) % 3;
            canvas.drawCircle(mPoints[m][n].x, mPoints[m][n].y, circleRadius, mPaint);
        }
    }
    
    public void drawLines(Canvas canvas, float curX, float curY)
    {
        mPaint.setStrokeWidth(centerRadius / 2);
        PointF lastPointF = null;
        for(int i = 0; i < pathNodes.size(); i++)
        {
            int m = pathNodes.get(i) / 3;
            int n = pathNodes.get(i) % 3;
            PointF curPointF  = mPoints[m][n];
            if(null != lastPointF)
            {
                canvas.drawLine(lastPointF.x, lastPointF.y, curPointF.x, curPointF.y, mPaint);
            }
            lastPointF = curPointF;
        }
        if(null != lastPointF)
        {
            canvas.drawLine(lastPointF.x, lastPointF.y, curX, curY, mPaint);
        }
    }
    
    public void detectGetPoint(float x, float y)
    {
        for(int i = 0; i < 3; i++)
        {
            for(int j = 0; j < 3; j++)
            {
                if((mPoints[i][j].x - x) * (mPoints[i][j].x - x) 
                        + (mPoints[i][j].y - y) * (mPoints[i][j].y - y) 
                        < centerRadius * centerRadius * 4) 
                {
                    /*触点进入某一中心点范围, 半径平方 乘以4 触点更大  容易操作*/
                    int nodeNum =  i * 3 + j;
                    if(!pathNodes.contains(nodeNum))
                    {
                        pathNodes.add(nodeNum);
                    }
                    return;
                }
                    
                
            }
        }
    }
    
    public String pathToString(List<Integer> list)
    {
        String des = "";
        for(int i = 0; i < list.size(); i++)
        {
            des += list.get(i).toString();
        }
        return des;
    }
    
    //设置完成事件监听回调
    public void setOnCompleteListener(OnCompleteListener o)
    {
        this.onCompleteListener = o;
    }
    
    public interface OnCompleteListener
    {
        public void onComplete(String pass);
    }
}

```
代码里的注释应该是比较清晰的。主要思路就是, 依次将9个点映射为0到8的整型。在手指滑动的过程中，用一个Integer类型的List保存滑过的点，并且通过PostInvalidate方法不断调用onDraw()进行重绘。包括9个点的绘制、滑过点周围圆圈的绘制以及点之间（包括手指现在的触点）连线的绘制。

同时编写了一个内部接口OnCompleteListener。自定义View中有该接口的对象， 由setOnCompleteListener()方法传入。在连接了3个或3个以上圆点的时候，手指抬起，调用其中的回调方法onComplete(String )。其中传入的String即为各选中点映射的整型组成的字符串。外部可对该字符串进行处理，或设置，或比对。

## 使用及Demo
使用方法很简单，直接在布局文件里添加就好了。
布局文件：activity_locus.xml

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <com.example.locusexample.LocusPassView
        android:id="@+id/locusview" 
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_marginTop="50dp"
        android:layout_marginBottom="50dp"/>
</LinearLayout>
```
设置密码的Activity: LocusSetActivity.java

```java
package com.example.locusexample;

import com.example.locusexample.LocusPassView.OnCompleteListener;

import android.app.Activity;
import android.content.Context;
import android.content.SharedPreferences;
import android.os.Bundle;
import android.widget.Toast;

public class LocusSetActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // TODO Auto-generated method stub
        super.onCreate(savedInstanceState);
        this.setContentView(R.layout.activity_locus);
        final SharedPreferences sp = this.getSharedPreferences("data", Context.MODE_PRIVATE);
        LocusPassView locusView = (LocusPassView) this.findViewById(R.id.locusview);
        locusView.setOnCompleteListener(new OnCompleteListener(){

            @Override
            public void onComplete(String pass) {
                // TODO Auto-generated method stub
                Toast.makeText(LocusSetActivity.this, "已设置密码：" + pass, 3000).show();
                sp.edit().putString("password", pass).commit();
                LocusSetActivity.this.finish();
            }
            
        });
    }

}

```
进行解锁的Activity: LocusUnlockActivity.java

```java
package com.example.locusexample;

import com.example.locusexample.LocusPassView.OnCompleteListener;

import android.app.Activity;
import android.content.Context;
import android.content.SharedPreferences;
import android.os.Bundle;
import android.widget.Toast;

public class LocusUnlockActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // TODO Auto-generated method stub
        super.onCreate(savedInstanceState);
        this.setContentView(R.layout.activity_locus);
        final SharedPreferences sp = this.getSharedPreferences("data", Context.MODE_PRIVATE);
        final String realPass = sp.getString("password", null);
        if(null == realPass)
        {
            Toast.makeText(this, "请先设置密码", 3000).show();
            this.finish();
        }
        LocusPassView locusView = (LocusPassView) this.findViewById(R.id.locusview);
        locusView.setOnCompleteListener(new OnCompleteListener(){

            @Override
            public void onComplete(String pass) {
                // TODO Auto-generated method stub
                Toast.makeText(LocusUnlockActivity.this, "输入密码：" + pass, 3000).show();
                if(pass.equals(realPass))
                {
                    Toast.makeText(LocusUnlockActivity.this, "密码正确", 3000).show();
                }
                else
                {
                    Toast.makeText(LocusUnlockActivity.this, "密码错误", 3000).show();
                }
            }
            
        });
    }

}

```
[点击下载完整的demo工程（即效果图所示）](http://download.csdn.net/detail/u013015161/8852407)