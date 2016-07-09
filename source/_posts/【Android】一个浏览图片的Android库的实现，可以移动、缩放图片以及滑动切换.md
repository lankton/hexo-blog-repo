---
title: 【Android】一个浏览图片的Android库的实现，可以移动、缩放图片以及滑动切换
date: 2016-07-09 21:14:36
categories: Lan's tech
tags:
  - Android
---

## 介绍
最近写了一个Library， 用于实现在Android设备上对大图的浏览。已经实现的功能有：
1、移动、缩放图片
2、双击快速放大或缩小图片
3、单击退出浏览
4、左右滑动切换图片。
目前还只实现了展示SD卡里图片的功能，后续应该补完，使其可以展示网络图片等。
代码已经在Github上开源， 地址为：
https://github.com/lankton/lanimagebrowser

展示：
图片切换  
<img src="http://img.blog.csdn.net/20150625192840175" width="200px"/>

图片缩放  
<img src="http://img.blog.csdn.net/20150625193024601" width="200px"/>

## 实现
实现的思路很简单。图片的缩放、移动等操作通过自定义ImageView实现，这些自定义ImageView通过Fragment来展现。同时，这些Fragment被绑定到ViewPager上，从而实现对图片的切换。下面简单讲一下几个比较关键的地方。
 1. 自定义ImageView
 主要重写了OnTouchEvent，来监听各种手势事件。同时重写了OnMeasure和OnLayout，来初始化图片在ImageView的显示。直接上代码吧。
 

```java
package com.lankton.imagebrowser;

import java.util.Timer;
import java.util.TimerTask;

import android.app.Activity;
import android.content.Context;
import android.graphics.Matrix;
import android.graphics.PointF;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.widget.ImageView;

public class BrowserImageView extends ImageView {

    Context context;
    
    float originDistance;
    float curDistance;
    float scale; //在上次基础上缩放
    float curScale = 1;
    float beginZoomScale; //开始缩放时的scale
    
    Matrix matrix = new Matrix();
    Matrix savedMatrix = new Matrix();
    PointF curPoint = new PointF();
    PointF lastPoint = new PointF();
    public BitmapSize bitmapSize;
    
    private Timer closeTimer;
    private boolean isClose;
    private final float BOUNDS = 30;
    private float originX;
    private float originY;
    
    float smallScale;
    float bigScale;
    boolean isToBig = true;
    
    public BrowserImageView(Context context, AttributeSet attrs) {
        super(context, attrs);
        this.context = context;
//        bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.sb);  
        // TODO Auto-generated constructor stub
    }
    
    public BrowserImageView(Context c)
    {
        super(c);
        this.context = c;
    }
    
    public void setBitmapSize(BitmapSize b)
    {
        this.bitmapSize = b;
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        // TODO Auto-generated method stub
        switch(event.getAction() & MotionEvent.ACTION_MASK)
        {
        case MotionEvent.ACTION_DOWN:
            curPoint.x = event.getX();
            curPoint.y = event.getY();
            savedMatrix.set(matrix);  
            isClose = true;
            originX = curPoint.x;
            originY = curPoint.y;
            break;
        case MotionEvent.ACTION_POINTER_DOWN:
            isClose = false;
            originDistance = getDistance(event.getX(0), event.getY(0),
                    event.getX(1), event.getY(1));
            beginZoomScale = curScale;
            break;
        case MotionEvent.ACTION_MOVE:
            if(isOutBounds(originX, originY, event.getX(0), event.getY(0)))
            {
                isClose = false;
                
            }
            
            if(event.getPointerCount() == 2)
            {
                curDistance = getDistance(event.getX(0), event.getY(0),
                        event.getX(1), event.getY(1));
                scale = curDistance / originDistance;
                curScale = beginZoomScale * scale;
                matrix.set(savedMatrix); 
                matrix.postScale(scale,  scale
                        , (event.getX(0) + event.getX(1))/2
                        , (event.getY(0) + event.getY(1))/2);
                this.setImageMatrix(matrix);
            }
            else if(event.getPointerCount() == 1)
            {
                lastPoint.x = curPoint.x;
                lastPoint.y = curPoint.y;
                curPoint.x = event.getX();
                curPoint.y = event.getY();
                matrix.postTranslate(curPoint.x - lastPoint.x, curPoint.y - lastPoint.y);
                this.setImageMatrix(matrix); 
            }
            break;
        case MotionEvent.ACTION_UP:
            if(isClose)
            {
                if(null == closeTimer)
                {
                    closeTimer = new Timer();
                    TimerTask task = new TimerTask(){

                        @Override
                        public void run() {
                            // TODO Auto-generated method stub
                            ((Activity) context).finish();
                        }
                        
                    }; 
                    closeTimer.schedule(task, 500);
                }
                else
                {//double click
                    closeTimer.cancel();
                    closeTimer = null;
                    if(isToBig)
                    {
                        matrix.postScale(bigScale / curScale , bigScale / curScale, event.getX(), event.getY());
                        curScale = bigScale;
                        this.setImageMatrix(matrix); 
                        isToBig = false;
                    }
                    else
                    {
                        matrix.postScale(smallScale / curScale, smallScale / curScale, event.getX(), event.getY());
                        curScale = smallScale;
                        this.setImageMatrix(matrix); 
                        isToBig = true;
                    }
                    
                }
                
            }
            break;
        case MotionEvent.ACTION_POINTER_UP:
            if(1 == event.getActionIndex())
            {
                curPoint.x = event.getX(0);
                curPoint.y = event.getY(0);
            }
            else
            {
                curPoint.x = event.getX(1);
                curPoint.y = event.getY(1);
            }
            savedMatrix.set(matrix);
            break;
        default:
            break;
        }
        return true;
    }
    
  
    /*获得两点间距离*/
    public float getDistance(float x1, float y1, float x2, float y2)
    {
        float disX = x1 - x2;
        float disY = y1 - y2;
        return (float)Math.sqrt(disX * disX + disY * disY);
    }
    
    /*设置图片以合适大小居中*/
    public void center()
    {
        int viewWidth = this.getMeasuredWidth();
        int viewHeight = this.getMeasuredHeight();
        int bitmapWidth = bitmapSize.width;
        int bitmapHeight = bitmapSize.height;
        float scale = 1;
        //先居中
        matrix.setTranslate((viewWidth - bitmapWidth)/2f, (viewHeight - bitmapHeight)/2f);
        
        //图片宽高有大于容器, 则需要再进行一次缩放处理
        if(bitmapWidth > viewWidth || bitmapHeight > viewHeight)
        {
            if((float)bitmapWidth / bitmapHeight > (float)viewWidth / viewHeight)
            {
                //宽、高比大于容器，以宽占满容器宽度为准，进行缩放
                scale = (float)viewWidth / bitmapWidth;               
                matrix.postScale(scale, scale, (float)viewWidth / 2, (float)viewHeight / 2);
            }
            else
            {
                //高、宽比大于容器， 以高占满容器高度为准，进行缩放
                scale = (float)viewHeight / bitmapHeight;
                matrix.postScale(scale, scale, (float)viewWidth / 2, (float)viewHeight / 2);
            }
        }
        smallScale = scale;
        bigScale = smallScale * 2;
        this.setImageMatrix(matrix);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // TODO Auto-generated method stub
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        refresh();
//        center();
        
    }
    @Override  
    protected void onLayout(boolean changed, int l, int t, int r, int b) {  
        if(null == bitmapSize)
        {
            return;
        }
        center();
    }  
    
    /*刷新matrix*/
    public void refresh()
    {
        matrix.reset();
        this.setImageMatrix(matrix);
    }
    
    public class BitmapSize
    {
        public int width;
        public int height;
        
        public BitmapSize(int width, int height)
        {
            this.width = width;
            this.height = height;
        }
    }
    /*手指在屏幕上移动超过范围才被判定为滑动，否则影响点击事件的判断*/
    public boolean isOutBounds(float x1, float y1, float x2, float y2 )
    {
        return Math.abs(x2 - x1) *  Math.abs(x2 - x1) 
                +  Math.abs(y2 - y1) * Math.abs(y2 - y1) > BOUNDS * BOUNDS; 
    }
    
}

```
可以看到，图片的位移及大小变换是通过修改matrix实现的，所以使用时该自定义View的scaleType被设为“matrix”。
 2. Fragment编写
 这个，也还是直接上代码吧。。
 

```java
package com.lankton.imagebrowser;

import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.util.Timer;
import java.util.TimerTask;

import com.lankton.imagebrowser.BrowserImageView.BitmapSize;


import android.R;
import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.os.AsyncTask;
import android.os.Bundle;
import android.support.v4.app.Fragment;
import android.support.v4.util.LruCache;
import android.util.Log;
import android.view.LayoutInflater;
import android.view.View;
import android.view.View.OnClickListener;
import android.view.ViewGroup;
import android.widget.ImageView.ScaleType;
import android.widget.RelativeLayout;
import android.widget.TextView;
import android.widget.Toast;
public class IFragment extends Fragment{

    public BrowserImageView img;
    private String path;
    private int pos;
    private Timer clickTimer;
    
    LruCache<String, Bitmap> cache;
    @Override
    public void onCreate(Bundle savedInstanceState) {
        // TODO Auto-generated method stub
        super.onCreate(savedInstanceState);
    }
    
    public IFragment()
    {
        super();
    }
    public IFragment(String path, int pos, LruCache<String, Bitmap> cache) {
        super();
        this.path = path;
        this.pos = pos;
        this.cache = cache;
    }

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
            Bundle savedInstanceState) {
        // TODO Auto-generated method stub
        Log.v("browser", ""+pos+" onCreateview");
        RelativeLayout relativeLayout = new RelativeLayout(this.getActivity());
        relativeLayout.setBackgroundColor(this.getResources().getColor(R.color.black));
        BrowserImageView bimg = new BrowserImageView(this.getActivity());
        RelativeLayout.LayoutParams lp = new RelativeLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT);
        lp.addRule(RelativeLayout.CENTER_IN_PARENT, RelativeLayout.TRUE);
        bimg.setScaleType(ScaleType.MATRIX);
        bimg.setClickable(true);
        relativeLayout.addView(bimg, lp);
        img = bimg;
        img.setOnClickListener(new OnClickListener(){

            @Override
            public void onClick(View v) {
                // TODO Auto-generated method stub
                Toast.makeText(IFragment.this.getActivity(), "click", 3000).show();
                if(null == clickTimer)
                {
                    clickTimer = new Timer();
                    TimerTask task = new TimerTask(){

                        @Override
                        public void run() {
                            // TODO Auto-generated method stub
                            IFragment.this.getActivity().finish();
                            Toast.makeText(IFragment.this.getActivity(), "close", 3000).show();
                        }
                        
                    }; 
                    clickTimer.schedule(task, 200);
                }
                else
                {
                    clickTimer.cancel();
                    clickTimer = null;
                }
                
                
            }
            
        });
        
        TextView t = new TextView(this.getActivity());
        t.setText("" + pos);
        t.setTextSize(30);
        t.setTextColor(this.getResources().getColor(R.color.white));
        relativeLayout.addView(t,lp);
        return relativeLayout;
    }

    @Override
    public void onDestroy() {
        // TODO Auto-generated method stub
        Log.v("browser", ""+pos+" onDestroy");
        super.onDestroy();
    }

    @Override
    public void onPause() {
        // TODO Auto-generated method stub  
        super.onPause();
    }

    @Override
    public void onResume() {
        // TODO Auto-generated method stub
        Log.v("browser", ""+pos+" onResume");
        super.onResume();
        setBitmap();
    }

    @Override
    public void onStop() {
        // TODO Auto-generated method stub
        Log.v("browser", ""+pos+" onStop");
        super.onStop();
        
    }
    
    public void setBitmap() {
        Bitmap bitmap = IFragment.this.getBitmapFromMemCache(path);
        if(null == bitmap)
        {
            LoadBitmapTask task = new LoadBitmapTask();
            task.execute();
        } else {
            BitmapSize bs = img.new BitmapSize(bitmap.getWidth(), bitmap.getHeight());
            img.setBitmapSize(bs);
            img.setImageBitmap(bitmap);
        }
        
    }
    
    public void addBitmapToMemoryCache(String key, Bitmap bitmap) {
        if (getBitmapFromMemCache(key) == null) {
            cache.put(key, bitmap);
        }
    }

    public Bitmap getBitmapFromMemCache(String key) {
        return cache.get(key);
    }
    
    class LoadBitmapTask extends AsyncTask<Void, Void, Bitmap>
    {

        @Override
        protected Bitmap doInBackground(Void... params) {
            // TODO Auto-generated method stub
            Bitmap bitmap = null;
            try {
                FileInputStream fin = new FileInputStream(path);
                final BitmapFactory.Options options = new BitmapFactory.Options();
                options.inSampleSize = 2;
                options.inJustDecodeBounds = false;
                bitmap = BitmapFactory.decodeStream(fin, null, options);
                IFragment.this.addBitmapToMemoryCache(path, bitmap);
            } catch (FileNotFoundException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            } 
            return bitmap;
        }

        @Override
        protected void onPostExecute(Bitmap result) {
            // TODO Auto-generated method stub
            super.onPostExecute(result);
            BitmapSize bs = img.new BitmapSize(result.getWidth(), result.getHeight());
            img.setBitmapSize(bs);
            img.setImageBitmap(result);
        }
        
    }
    
}

```
由于工程要被拿来当作library，可以看到在onCreateView通过代码生成自定义的BrowserImageView并被设置到布局里。BrowserImageView的scaleType被设置成“matrix”。加载时，通过AsyncTask异步加载图片。
 
 3. 图片缓存
通过LruCache动态进行内存管理，否则很高概率出现OOM。缓存初始化放在了ViewPager的Adapter里：

```java
package com.lankton.imagebrowser;

import java.util.List;

import android.graphics.Bitmap;
import android.support.v4.app.Fragment;
import android.support.v4.app.FragmentManager;
import android.support.v4.app.FragmentPagerAdapter;
import android.support.v4.util.LruCache;
import android.util.Log;

public class IPagerAdapter extends FragmentPagerAdapter{
    List<String> pathList;
    LruCache<String, Bitmap> cache;
    
    public IPagerAdapter(FragmentManager fm, List<String> pathList) {
        super(fm);
        this.pathList = pathList;
        
        /*init LruCache*/
        final int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);Log.v("diskcache","mem : " + maxMemory);
        final int cacheSize = 1 * 1024 * 1024;//maxMemory / 8;
        cache = new LruCache<String, Bitmap>(cacheSize);
        // TODO Auto-generated constructor stub
    }

    @Override
    public Fragment getItem(int position) {
        // TODO Auto-generated method stub
        IFragment f = new IFragment(pathList.get(position),position, cache);
        return f;
    }

    @Override
    public int getCount() {
        // TODO Auto-generated method stub
        return pathList.size();
    }

}

```
在之前介绍Fragment的代码里可以看到如何使用LruCache的，不再赘述了。

## 使用
本Library主要提供了一个PagerAdapter。使用时，让该工程作为Library被需要的工程引用即可。
使用时的代码如下： 

```
@Override
    protected void onCreate(Bundle savedInstanceState) {
        // TODO Auto-generated method stub
        super.onCreate(savedInstanceState);
        this.setContentView(R.layout.lanactivity_browserimage);
        viewPager = (ViewPager) this.findViewById(R.id.viewpager);
        
        
        
        photos = this.getIntent().getStringArrayListExtra("photoList");
        index = this.getIntent().getIntExtra("index", 0);
        
        adapter = new IPagerAdapter(this.getSupportFragmentManager(), photos);
        viewPager.setAdapter(adapter);
        viewPager.setCurrentItem(index);
        
        
    }
```
在你的布局文件里放置一个普通的ViewPager，然后使用类库提供的IPagerAdapter即可。需要传进Adapter的参数，即photos，是你本地文件的路径列表。之后该Activity就可以拿来进行图片浏览了。

就先介绍到这里吧， 这个Library目前还有不少需要改进和提升的地方，请多多指教。

最后在申明一次开源地址， 代码都可以从这里获取：
https://github.com/lankton/lanimagebrowser

## 更新
###解决viewpager和imageview的滑动冲突 2015 6 28###
之前版本，想直接左右拖动图片时(eg 图片放大状态，想查看未显示的部分)，会直接出发viewpager的翻页事件。
解决方案：手指在imageview上move时，根据条件判断是否应该禁止viewpager的滑动事件。参考链接：[requestDisallowInterceptTouchEvent](http://www.cnblogs.com/xitang/archive/2013/06/22/3150380.html)
参考里viewpager直接传递进子view， 其实不用，可以直接通过getParent()获得。同时本library的情况要分别考虑左划和右划。代码如下：

```java
Rect rectTemp = this.getDrawable().getBounds(); 
                matrix.getValues(values);
                int leftPos = (int)values[2];
                int rightPos = (int)(values[2]+rectTemp.width()*values[0]);  
                lastPoint.x = curPoint.x;
                lastPoint.y = curPoint.y;
                curPoint.x = event.getX();
                curPoint.y = event.getY();
                if(leftPos < 0 && curPoint.x > lastPoint.x || rightPos > viewWidth && curPoint.x < lastPoint.x)
                {// 图片左边未显示完全时禁止向右划，向左划同理
                    this.getParent().requestDisallowInterceptTouchEvent(true);
                }
                else 
                {
                    this.getParent().requestDisallowInterceptTouchEvent(false);
                    return true;
                }
```
已同步至git。