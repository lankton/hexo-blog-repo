---
title: 【Android】0行代码实现任意形状图片展示--android-anyshape
date: 2016-07-09 22:01:58
categories: Lan's tech
tags:
  - Android
---
# 前言
在Android开发中， 我们经常会遇到一些场景， 需要以一些特殊的形状显示图片， 比如圆角矩形、圆形等等。关于如何绘制这类形状， 网上已经有很多的方案，比如自定义控件重写onDraw方法， 通过canvas的各种draw方法进行绘制等。那么， 更复杂的图形呢？比如，五角星？比如组合图形？又或者是各种奇奇怪怪的不规则图形呢？有同学会说， 如果已知不规则图形的具体形状， 那我们就可以通过连接顶点的方式， 找出path， 然后通过drawPath方法绘制出来啊。嗯。。。很有道理， 但是先不说有些图像，可能顶点巨多， 或者弯弯曲曲很难找出具体的顶点， 难道我们要为每一个特殊的形状， 单独写一个独立的控件， 或者一套独立的代码吗？
可以肯定是可以，但是我觉得， 最好还是不要这么做。。于是我有了一个想法， **用一张图片， 告诉控件，我想要什么样的形状， 然后控件自动按照这个形状， 帮我把图片显示出来**。于是有了这个项目－－[android-anyshape](https://github.com/lankton/android-anyshape)。

# 展示
<img src="http://img.blog.csdn.net/20160327163538291" width="200px"/>&nbsp;&nbsp;&nbsp;<img src="http://img.blog.csdn.net/20160327163638432" width="200px"/>  
左边是使用了普通ImageView的展示效果， 右边是使用了项目中AnyshapeImageView的效果。想使用AnyshapeImageView达到右边的样式， 仅需提供三张遮罩图片，通过"anyshapeMask"参数提供给控件即可（下文会说明）。
三张“遮罩”图片如下：  
<img src="http://img.blog.csdn.net/20160327164339919" width="150px"/>&nbsp;&nbsp;&nbsp;<img src="http://img.blog.csdn.net/20160327175945131" width="150px"/>&nbsp;&nbsp;&nbsp;<img src="http://img.blog.csdn.net/20160327180054263" width="150px"/>  
与普通的遮罩图片不同， 这里要求**图片的背景完全透明， 即alpha通道的值为0， 而需要显示的图形，对具体的颜色没有任何要求，不透明即可**。
# 使用
控件的使用很简单， 由于继承ImageView， 所以使用方法类似于ImageView，但多了一个重要的自定义参数：anyshapeMask
```xml
<cn.lankton.anyshape.AnyshapeImageView
   android:layout_width="150dp"
   android:layout_height="150dp"
   android:layout_marginTop="20dp"
   android:src="@drawable/kumamon"
   app:anyshapeMask="@drawable/singlestar"/>
```
在布局文件中加入这段xml， 展示的就是上面图中那头五角星形状的熊本熊～
实现这个功能的思路其实很简单，通过对一张“遮罩”图片各像素透明度的扫描，获得一个Path对象， 该Path对象包含了所有不透明像素的集合。然后就很简单了， 通过Canvas对象的drawPath方法，将我们要显示的图片刷上去即可。
# 实现
## 从Bitmap中提取Path
**这是这个项目中最重要的部分**。代码如下：
PathInfoManager.getPathFromBitmap:
```java
public Path getPathFromBitmap(Bitmap mask) {
    Path path = new Path();
    int bWidth = mask.getWidth();
    int bHeight = mask.getHeight();
    int[] origin = new int[bWidth];
    int lastA;
    for (int i = 0; i < bHeight; i++) {
        mask.getPixels(origin, 0, bWidth, 0, i, bWidth, 1);
        lastA = 0;
        for (int j = 0; j < bWidth; j++) {
            int a = Color.alpha(origin[j]);
            if (a != 0 && lastA == 0) {
                path.moveTo(j, i);
            } else if (a == 0 && lastA !=0 ) {
                path.lineTo(j - 1, i);
            } else if (a != 0 && j == bWidth - 1) {
                path.lineTo(j, i);
            }
            lastA = a;
        }
    }
    return path;
}
```
我设计的方案很简单，逐行扫描Bitmap中的像素，实现方法是用getPixels方法获得每行的像素数组，然后遍历分析。步骤如下：
1. 遇到一个不透明像素，进行判断， 如果它的上一个像素不透明， 或者它本身就是行首， 那我们就把它看作一段不透明区域的开头，通过moveTo方法将Path移动到此点；
2. 遇到一个透明像素，进行判断，如果它的上一个像素透明，那我们就把它的上一个像素看作一段不透明区域的结尾， 通过lineTo的方式， 将它与之前的开头像素连接。
3. 重复1、2步， 直到扫描完全行。需要注意的是， 如果行尾是不透明像素， 那就直接连上。防止最后一段不透明区域只有起点没有终点。
这样， 每一行的连接结果，就组成了整张图片的扫描结果～
## 通过Path，显示图像
先看一下AnyshapeImageView的初始化方法：

```java
public AnyshapeImageView(Context context, AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
    this.context = context;
    TypedArray a = context.getTheme().obtainStyledAttributes(attrs, R.styleable.AnyShapeImageView, defStyleAttr, 0);
    int n = a.getIndexCount();
    for (int i = 0; i < n; i++)
    {
        final int attr = a.getIndex(i);
        if (attr == R.styleable.AnyShapeImageView_anyshapeMask) {
            maskResId = a.getResourceId(attr, 0);
            if (0 == maskResId) {
                //did not set mask
                continue;
            }

        } else if (attr == R.styleable.AnyShapeImageView_anyshapeBackColor) {
            backColor = a.getColor(attr, Color.TRANSPARENT);
        }
    }
    a.recycle();
}
```
其实就是调用通过anyshapeMask参数， 获得“遮罩”图片的资源ID以及背景色。 真正通过资源ID解析获取遮罩的过程放在了onMeaaure中。
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    int mWidth = getMeasuredWidth();
    int mHeight = getMeasuredHeight();
    if (mWidth != 0 && mHeight != 0) {
        if (maskResId <= 0) {
            return;
        }
        PathInfo pi = PathManager.getInstance().getPathInfo(maskResId);
        if (null != pi) {
            originMaskPath = pi.path;
            originMaskWidth = pi.width;
            originMaskHeight = pi.height;
        } else {
            BitmapFactory.Options options = new BitmapFactory.Options();
            options.inJustDecodeBounds = true;
            BitmapFactory.decodeResource(context.getResources(), maskResId, options);
            int widthRatio = (int)(options.outWidth * 1f / mWidth);
            int heightRatio = (int)(options.outHeight * 1f / mHeight);
            if (widthRatio > heightRatio) {
                options.inSampleSize = widthRatio;
            } else {
                options.inSampleSize = heightRatio;
            }
            if (options.inSampleSize == 0) {
                options.inSampleSize = 1;
            }
            options.inJustDecodeBounds = false;
            Bitmap maskBitmap = BitmapFactory.decodeResource(context.getResources(), maskResId, options);
            originMaskPath = PathManager.getInstance().getPathFromBitmap(maskBitmap);
            originMaskWidth = maskBitmap.getWidth();
            originMaskHeight = maskBitmap.getHeight();
            pi = new PathInfo();
            pi.height = originMaskHeight;
            pi.width = originMaskWidth;
            pi.path = originMaskPath;
            PathManager.getInstance().addPathInfo(maskResId, pi);
            maskBitmap.recycle();
        }
    }
}
```
PathInfo：
```java
public class PathInfo {
    public Path path;
    public int width;
    public int height;
}
```
然而我们看到，用户进行生成Bitmap－获取Path这一系列耗时、耗内存操作之前，先会判断缓存里是否已经有与该资源ID匹配的PathInfo， 如果有， 则不用进行这部分操作。如果没有，根据传入的资源ID，生成PathInfo对象，并存入缓存。同时，根据控件的宽高，对decode做了限制，预防了OOM 和 加载资源过大的问题。
关于这块缓存，下面会说明。

再看onSizeChanged方法：

```java
protected void onSizeChanged(int w, int h, int oldw, int oldh) {
    super.onSizeChanged(w, h, oldw, oldh);
    vHeight = getHeight();
    vWidth = getWidth();
    if (originMaskPath != null) {
        //scale the size of the path to fit the one of this View
        Matrix matrix = new Matrix();
        matrix.setScale(vWidth * 1f / originMaskWidth, vHeight * 1f / originMaskHeight);
        originMaskPath.transform(matrix, realMaskPath);
    }
}
```
这里的代码， 主要目的是对Path对象进行缩放， 已匹配控件的实际大小。可以看到， **如果不希望展示的形状被拉伸或者变形， 那么AnyshapeImageView的宽高比， 最好和“遮罩”图片的宽高比保持一致。**

接下来就是在onDraw里绘制形状并刷上图片了：

```java
@Override
protected void onDraw(Canvas canvas) {
    if (null == originMaskPath) {
        // if the mask is null, the view will work as a normal ImageView
        super.onDraw(canvas);
        return;
    }
    if (vWidth == 0 || vHeight == 0) {
        return;
    }

    paint.reset();
    paint.setStyle(Paint.Style.STROKE);
    //get the drawable to show. if not set the src, will use  backColor
    Drawable showDrawable = getDrawable();
    if (null != showDrawable) {
        Bitmap showBitmap = ((BitmapDrawable) showDrawable).getBitmap();
        Shader shader = new BitmapShader(showBitmap, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
        Matrix shaderMatrix = new Matrix();
        float scaleX = vWidth * 1.0f / showBitmap.getWidth();
        float scaleY = vHeight * 1.0f / showBitmap.getHeight();
        shaderMatrix.setScale(scaleX, scaleY);
        shader.setLocalMatrix(shaderMatrix);
        paint.setShader(shader);
    } else {
        //no src , use the backColor to fill the path
        paint.setColor(backColor);
    }
    canvas.drawPath(realMaskPath, paint);

}
```
## 缓存
看了上面的博文， 各位一定清楚了，作为参数传入的资源ID，实际上只是为了获取一个Path对象。那么我们可以建立一个Integer－Path的映射关系， 用来缓存已经读取出来的Path。后面需要Path， 只需要通过资源ID去缓存里寻找即可，毕竟读取Path是一个费时间又费资源的操作。
这样看来，我们已经对AnyshapeImageView的使用进行了优化， 毕竟同一个形状的展示，我们只要执行一次从图片中解析Path对象的操作即可。

# 总结
这个项目，是我花了将近一周时间断断续续完成的。代码不多， 也不复杂，希望能够帮到大家， 或者为大家提供一些思路。
再贴一下项目的地址， 包括demo在内：
[https://github.com/lankton/android-anyshape](https://github.com/lankton/android-anyshape)
如果你觉得这个项目，或者这篇博文对你起到了一些帮助，欢迎star支持一下～ 

# 更新
## 2016-05-12
优化了AnyshapeImageView解析遮罩的过程，PathManager中的createPaths（预先解析Path）变得繁琐且不必要，故删除。 简化后的使用可见[项目README](https://github.com/lankton/android-anyshape/blob/master/README.md)。
本次更新后对博文上面代码、讲解内容也有改动。
## 发布到JCenter-20160519
为方便使用，已将library发布到JCenter，开发者可以使用gradle或者maven进行依赖的配置。
###latest version
可见[项目README](https://github.com/lankton/android-anyshape/blob/master/README.md)头部图标

### gradle
```
compile 'cn.lankton:anyshape:latest version'
```
### maven
```
<dependency>
  <groupId>cn.lankton</groupId>
  <artifactId>anyshape</artifactId>
  <version>latest version</version>
  <type>pom</type>
</dependency>
```
