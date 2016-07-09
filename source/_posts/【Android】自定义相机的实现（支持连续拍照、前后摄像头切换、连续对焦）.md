---
title: 【Android】自定义相机的实现（支持连续拍照、前后摄像头切换、连续对焦）
date: 2016-07-09 21:36:52
categories: Lan's tech
tags:
  - Android
---
# 介绍
这几天，写了一个自定义照相机的demo，支持连续拍照和摄像头切换。由于自己以前没接触过相关的编程，也算是一个学习的过程，在这里做一下记录，同时也分享出来，并附上源码和工程。
效果如图：
![这里写图片描述](http://img.blog.csdn.net/20150717011532595)
左上角switch切换摄像头，右边snap按钮进行拍照。

# 一般流程
Android进行拍照，需要调用摄像头类android.hardware.Camera。而要进行预览，则需要用android.view.SurfaceView对每一帧的预览图进行显示。
实现自定义相机一般流程为：
1、用addCallback给SurfaceView设置Callback接口对象，实现其中三个回调方法：surfaceCreated、surfaceChanged、surfaceDestroyed。
在surfaceCreated中打开摄像头，获得Camera对象，并设置其在surfaceview上预览；
在surfaceChanged中设置摄像头的参数；
在surfaceDestroyed释放摄像头，否则会导致退出之后其他应用无法调用摄像头，包括系统相机。
2、点击拍照按钮时，调用Camera对象的takePicture方法，其第三个参数为PictureCallback接口对象，其中的onPictureTaken回调方法参数中有一个byte数组，存储了拍摄到的图片数据，在方法中保存到本地即可。
这样，一个基本可用、带预览的自定义相机就做好了。但这样还远远不够，因为会出现各种各样的问题。

# 主要问题
## 预览变形
这个是最头疼的问题。首先要知道3个宽高比：摄像头分辨率（PictureSize）宽高比、预览分辨率（PreviewSize）宽高比以及用作预览的SurfaceView的宽高比。如果要让预览不变形，这三个宽高比需要保持一致。这种一致性的保持在设置摄像头参数时进行。代码如下：

```java
public void setCameraAndDisplay(int width, int height)
    {
        Camera.Parameters parameters = camera.getParameters();
        /*获取摄像头支持的PictureSize列表*/
        List<Camera.Size> pictureSizeList = parameters.getSupportedPictureSizes();
        /*从列表中选取合适的分辨率*/
        Size picSize = CameraUtils.getProperSize(pictureSizeList, ((float)width)/height);
        if(null != picSize)
        {
            parameters.setPictureSize(picSize.width, picSize.height);
        }
        else
        {
            picSize = parameters.getPictureSize();
        }
        /*获取摄像头支持的PreviewSize列表*/
        List<Camera.Size> previewSizeList = parameters.getSupportedPreviewSizes();
        Size preSize = CameraUtils.getProperSize(previewSizeList, ((float)width)/height);
        if(null != preSize)
        {Log.v("TestCameraActivityTag", preSize.width + "," + preSize.height);
            parameters.setPreviewSize(preSize.width, preSize.height);
        }
        
        /*根据选出的PictureSize重新设置SurfaceView大小*/
        float w = picSize.width;
        float h = picSize.height;
        surfaceView.setLayoutParams(new RelativeLayout.LayoutParams( (int)(height*(w/h)), height)); 
        
        parameters.setJpegQuality(100); // 设置照片质量  
        if (parameters.getSupportedFocusModes().contains(Camera.Parameters.FOCUS_MODE_CONTINUOUS_PICTURE))
        {
            parameters.setFocusMode(Camera.Parameters.FOCUS_MODE_CONTINUOUS_PICTURE);
        }
        
      
        
        camera.cancelAutoFocus();//只有加上了这一句，才会自动对焦。  
        camera.setDisplayOrientation(0);
        camera.setParameters(parameters);
    }
```

方法里传进去的参数为SurfaceView现在的宽高。在保证surfaceview宽高比不变的情况下（比如为了保证全屏预览），分别去寻找符合条件的PictureSize和PreviewSize。如果找不到，就返回默认宽高比（我设置为4:3）的PictureSize和PreviewSize， 同时为保证3个宽高比一致，surfaceView的宽高比也要重设。
提供寻找合适的PictureSize和PreviewSize方法的类如下：

```java
public class CameraUtils {
    
    public static Size getProperSize(List<Size> sizeList, float displayRatio)
    {
        //先对传进来的size列表进行排序
        Collections.sort(sizeList, new SizeComparator());
        
        Size result = null;
        for(Size size: sizeList)
        {
            float curRatio =  ((float)size.width) / size.height;
            if(curRatio - displayRatio == 0)
            {
                result = size;
            }
        }
        if(null == result)
        {
            for(Size size: sizeList)
            {
                float curRatio =  ((float)size.width) / size.height;
                if(curRatio == 3f/4)
                {
                    result = size;
                }
            }
        }
        return result;
    }
    
    static class SizeComparator implements Comparator<Size>
    {

        @Override
        public int compare(Size lhs, Size rhs) {
            // TODO Auto-generated method stub
            Size size1 = lhs;
            Size size2 = rhs;
            if(size1.width < size2.width 
                    || size1.width == size2.width && size1.height < size2.height)
            {
                return -1;
            }
            else if(!(size1.width == size2.width && size1.height == size2.height))
            {
                return 1;
            }
            return 0;
        }
        
    }
}

```


由于不同的手机返回的支持分辨率的排序不一样（我手中一款联想从小到大排序，而另一部nexus 4从大到小排序），所以需要先对列表统一进行从小到大排序。这样，方法返回的就是符合条件宽高比的最大分辨率，可以保证照片的清晰度。

## 照片方向错误
解决方法是监听手机方向的改变。监听到方向发生变化，就调用Camera.Parameters 对象(Camera对象调用getParameters()方法获得)的setRotation方法重新设置成像方向。
监听的方法是，在Activity设置一个实现了OrientationEventListener接口 的对象，并调用其enable()方法激活。

```java
IOrientationEventListener iOriListener;
... ...
iOriListener.enalbe();
```

实现OrientationEventListener接口的类：

```java
public class IOrientationEventListener extends OrientationEventListener{

        public IOrientationEventListener(Context context) {
            super(context);
            // TODO Auto-generated constructor stub
        }


        @Override
        public void onOrientationChanged(int orientation) {
            // TODO Auto-generated method stub
            if(ORIENTATION_UNKNOWN == orientation)
            {
                return;
            }
            CameraInfo info = new CameraInfo();
            Camera.getCameraInfo(camera_id, info);
            orientation = (orientation + 45) / 90 * 90;
            int rotation = 0;
            if(info.facing == CameraInfo.CAMERA_FACING_FRONT)
            {
                rotation = (info.orientation - orientation + 360) % 360;
            }
            else
            {
                rotation = (info.orientation + orientation) % 360;
            }
            if(null != camera)
            {
                Camera.Parameters parameters = camera.getParameters();
                parameters.setRotation(rotation);
                camera.setParameters(parameters);
            }
            
        }
        
    }
```
其中设置rotation的公式由google官方提供。

## 预览方向错误
由于我设置了相机的Activity的方向为ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE，即横屏模式，所以没有遇到这个问题。
不过上一篇我转载的博文里，有提到相关问题的解决。相关代码摘录出来，供大家参考。以下代码放在surfaceview的callback中surfaceChanged方法里执行：

```java
if(this.getResources().getConfiguration().orientation != Configuration.ORIENTATION_LANDSCAPE)

{

//如果是竖屏

parameters.set(“orientation”, “portrait”);

//在2.2以上可以使用

//camera.setDisplayOrientation(90);

}

else

{

parameters.set(“orientation”, “landscape”);

//在2.2以上可以使用

//camera.setDisplayOrientation(0);

}

```
## 连续对焦
通过连续对焦，可以拍出清楚的照片。代码在上面的代码段里出现过

```java
/*先判断是否支持，否则可能报错*/
        if (parameters.getSupportedFocusModes().contains(Camera.Parameters.FOCUS_MODE_CONTINUOUS_PICTURE))
        {
            parameters.setFocusMode(Camera.Parameters.FOCUS_MODE_CONTINUOUS_PICTURE);
        }
        camera.cancelAutoFocus();//只有加上了这一句，才会自动对焦。  
```

照相机Activity的代码就不全贴了，更具体的可以参考demo工程。

# 工程下载

[点此下载自定义相机demo工程](http://download.csdn.net/download/u013015161/8907491)。

由于本人手中测试机型只有两台，运行了上述demo的朋友方便的话可以 告知下运行效果。