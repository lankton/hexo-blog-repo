---
title: 【Android】自定义LinearLayout实现侧滑布局--SwipeLinearLayout
date: 2016-07-09 22:21:42
categories: Lan's tech
tags:
  - Android
---
## 描述
这周做了一个自定义侧滑布局， 继承自LinearLayout。
代码地址：[android-SwipeLinearLayout](https://github.com/lankton/android-SwipeLinearLayout)
## 效果
可以单独使用，也可以在ListView等可滑动的父组件中使用。以在ListView中使用为demo:  
<img src="http://img.blog.csdn.net/20160525015953736" width="260px"/>  
解决了item和ListView的滑动冲突， 同时每个item及其上面的控件可以正常点击。

代码比较简单，就不上传到JCenter了。 控件本身就只有一个文件: [SwipeLinearLayout.java](https://github.com/lankton/android-SwipeLinearLayout/blob/master/app/src/main/java/cn/lankton/swipelinearlayout/lib/SwipeLinearLayout.java), 有需要可以直接复制或者修改。

## 使用
和普通LinearLayout一样使用，内部包含2个子元素即可。
示例:
```xml
<xx.SwipeLinearLayout
  xxxx>
  <LinearLayout
    android:layout-width="match_parent"
    xxxx
    xxxx>
    ... ...
  </LinearLayout>
  <LinearLayout
    android:layout-width="30dp"
    xxxx>
    ... ...
  </LinearLayout>
</xx.SwipeLinearLayout>
```
第一个子元素是未侧滑时就显示的部分， 第二个子元素是会被侧滑出来的部分。
**SwipeLinearLayout的orientation随便设置，反正都会当成horizontal处理。 **
```java
public SwipeLinearLayout(Context context) {
    this(context, null);
}

public SwipeLinearLayout(Context context, AttributeSet attrs) {
    this(context, attrs, 0);
}

public SwipeLinearLayout(Context context, AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
    mScroller = new Scroller(context);
    this.setOrientation(HORIZONTAL);
}
```

## 实现
### 如何进行滑动
这个问题思路很简单。滑动分为2个阶段， 一个阶段就是跟手滑动，另外一个阶段，就是当手指离开后，布局继续滑动。
跟手滑动，那么我们很容易就想到重写onTouchEvent方法，在ACTION_MOVE事件中实现。那手指离开之后呢？首先要明确一点，开始处理的判断，是放在ACTION_UP事件中的。我们可以通过此时布局展开的程度，决定布局是要完全展开，还是缩回初始状态。为了让这种自动的滚动显得自然，我们需要借助Scroller。
Scroller可以看作一种类似插值器一样的东西，可以在系统调用的回调中，为我们提供一个起、终值之间的值。随着时间的增长，这个值逐渐从起点值变成终点值。通过这个值随时间的变化，可以帮助我们实现布局的平滑滚动。
处理滑动的代码如下：
```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
        case MotionEvent.ACTION_POINTER_DOWN:
            lastX = event.getX();
            lastY = event.getY();
            startScrollX = getScrollX();
            break;
        case MotionEvent.ACTION_MOVE:
            if (ignore) {
                ignore = false;
                break;
            }
            float curX = event.getX();
            float dX = curX - lastX;
            lastX = curX;
            if (hasJudged) {
                int targetScrollX = getScrollX() + (int)(-dX);
                if (targetScrollX > width_right) {
                    scrollTo(width_right, 0);
                } else if (targetScrollX < 0) {
                    scrollTo(0, 0);
                } else {
                    scrollTo(targetScrollX, 0);
                }
            }
            break;
        case MotionEvent.ACTION_UP:
            float finalX = event.getX();
            if (finalX < startX) {
                scrollAuto(DIRECTION_EXPAND);
            }  else {
                scrollAuto(DIRECTION_SHRINK);
            }
            break;
        default:
            break;
    }
    return true;
}

/**
 * 自动滚动， 变为展开或收缩状态
 * @param direction
 */
public void scrollAuto(final int direction) {
    int curScrollX = getScrollX();
    if (direction == DIRECTION_EXPAND) {
        // 展开
        mScroller.startScroll(curScrollX, 0, width_right - curScrollX, 0, 300);
    } else {
        // 缩回
        mScroller.startScroll(curScrollX, 0, -curScrollX, 0, 300);
    }
    invalidate();
}


@Override
public void computeScroll() {
    super.computeScroll();
    if (mScroller.computeScrollOffset()) {
        this.scrollTo(mScroller.getCurrX(), 0);
        invalidate();
    }
}

```
可以看到我们就是在computeScroll()方法中，获得插值，进行滚动的。要注意的是，一定要调用invalidate()，computeScroll() 才会被调用。
关于hasJudged和ingnore标志位， 这两个是跟处理滑动冲突相关的。hasJudged标志位表示: 当前手指滑动的方向(水平or竖直)是否已经判断出，ignore表示是否要忽略这次被传到onTouchEvent里的事件。
我们继续往下看。
### 处理滑动冲突
处理滑动冲突的目的是，保证布局的左右滑动，和它父组件，如ListView等的竖直滑动，不会相互影响。如果仅仅像上文一样，只实现了onTouchEvent， 那么单独使用该布局，倒是没什么问题。但在ListView的item中使用的时候，你会发现，在你想划开子item的时候，很容易就引起了ListView的上下滑动。而且之后的所有事件， 都会被ListView拦截。这就很尴尬了，SwipeLinearLayout刚被划开一点就不动了。而且这种情况出现的非常频繁，滑动冲突必须处理，即：
**touch事件被谁处理，必须由我们说了算。**
本次处理滑动冲突，我采用的是内部拦截法。即，在子View的dispatchTouchEvent中，先使用父View的requestDisallowInterceptTouchEvent(true)，阻止父View对后续事件进行拦截。然后再通过后续条件判断，是否让父View恢复拦截事件的能力。
在本例中，我们通过比较手指在水平方向和竖直方向移动距离的大小，判断是否调用requestDisallowInterceptTouchEvent(false)恢复父View拦截能力。为了判断更合理， 比较放在了手指移动超过一定距离的时候。
```java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    switch (ev.getActionMasked()) {
        case MotionEvent.ACTION_DOWN:
            disallowParentsInterceptTouchEvent(getParent());
            hasJudged = false;
            startX = ev.getX();
            startY = ev.getY();
            break;
        case MotionEvent.ACTION_MOVE:
            float curX = ev.getX();
            float curY = ev.getY();
            if (hasJudged == false) {
                float dx = curX - startX;
                float dy = curY - startY;
                if ((dx * dx + dy * dy > MOVE_JUDGE_DISTANCE * MOVE_JUDGE_DISTANCE)) {
                    if (Math.abs(dy) > Math.abs(dx)) {
                        allowParentsInterceptTouchEvent(getParent());
                        if (null != onSwipeListener) {
                            onSwipeListener.onDirectionJudged(this, false);
                        }
                    } else {
                        if (null != onSwipeListener) {
                            onSwipeListener.onDirectionJudged(this, true);
                        }
                        lastX = curX;
                        lastY = curY;
                    }
                    hasJudged = true;
                    ignore = true;
                }
            }
            break;
        case MotionEvent.ACTION_UP:
            break;
        default:
            break;
    }
    return super.dispatchTouchEvent(ev);
}
```
hasJudged， 很好理解，表示:当前手指滑动的方向(水平or竖直)是否已经判断出。那ignore呢，又是什么鬼？
是这样的：当我们判断手指其实是竖直方向滑动的时候，会恢复父View(如ListView)的拦截能力，那后续的滑动，其实都只是ListView的上下滑动了。这个，大家应该都能理解。但大家要注意一点，决定滑动方向的，最后一次ACTION_MOVE事件，依然被传到onTouchEvent里去了。这就会造成，虽然结果判定是对ListView进行上下滑动，但我们依然可以看见，相应的item的SwipeLinearLayout被划出来了一点。这就很难看了。于是我增加了一个ignore标志位，来表示，忽略这次的事件。即：**用来决定方向的手指滑动，就只是用来决定方向的，而不会对UI产生任何影响。**

你也可能发现，这里并没有直接调用parent的requestDisallowInterceptTouchEvent方法，而是调用了自定义的方法disallowParentsInterceptTouchEvent以及allowParentsInterceptTouchEvent。
看一下这两个方法：
```java
private void disallowParentsInterceptTouchEvent(ViewParent parent) {
    if (null == parent) {
        return;
    }
    parent.requestDisallowInterceptTouchEvent(true);
    disallowParentsInterceptTouchEvent(parent.getParent());
}

private void allowParentsInterceptTouchEvent(ViewParent parent) {
    if (null == parent) {
        return;
    }
    parent.requestDisallowInterceptTouchEvent(false);
    allowParentsInterceptTouchEvent(parent.getParent());
}
```
用了递归，原因很简单：你想阻止或者恢复拦截的，并不一定是SwipeLinearLayout的直接父组件。举个例子，SwipeLinearLayout可能只是你的item布局的一个子布局， 那它的父布局就不是ListView。我们要阻止ListView，就只能通过递归的方式，向上搜索，然后调用requestDisallowInterceptTouchEvent(false)。

### 处理点击事件
一个View被设置了OnClickListener，其onClick方法其实是在OnTouchEvent的ACTION_UP中调用的。所以SwipeLinearLayout的子控件，如果想点击事件生效，就必须得到事件。而为了保证SwipeLinearLayout的滑动，SwipeLinearLayout的又必须对事件进行拦截。所以，可以重写SwipeLinearLayout的处理拦截方法如下：
```java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    if (hasJudged) {
        return true;
    }
    return super.onInterceptTouchEvent(ev);
}
```
 应该很好理解，当判定了滑动方向的时候(其实就是水平方向， 如果是竖直方向的话，直接就被上层拦截了，到不了这里)，返回true， 自己消费touch事件，没判定的话，就返回父类，即LinearLayout的onInterceptTouchEvent。LinearLayou的子控件可以点击吗？当然可以。所以这样写就ok了。
看到的效果如示例动图：
拖动左边白色部分的时候，虽然手指一直在上面，也是从上面离开的，但依然不会出发click事件。但直接点击的话，则会弹出Toast，提示点击了item。

### ListView中item的联动
这个描述，说的其实就是图片里显示的，竖直滑动ListView，或者滑动其他的item， 已经展开的item会复原。
这个重点其实不在SwipeLinearLayout上了，具体的逻辑是在与ListView对应的Adapter上。
SwipeLinearLayout中提供了这样一个interface:
```java
public interface OnSwipeListener {
    /**
     * 手指滑动方向明确了
     * @param sll  拖动的SwipeLinearLayout
     * @param isHorizontal 滑动方向是否为水平
     */
    void onDirectionJudged(SwipeLinearLayout sll, boolean isHorizontal);
}
```
onDirectionJudged， 在hasJudged被置为true的时候被调用。在上面的代码中也可以看到。
下面看Adapter中是如何实现这个接口的：
```java
@Override
public void onDirectionJudged(SwipeLinearLayout thisSll, boolean isHorizontal) {
    if (false == isHorizontal) {
        for (SwipeLinearLayout sll : swipeLinearLayouts) {
            if (null == sll) {
                continue;
            }
            sll.scrollAuto(SwipeLinearLayout.DIRECTION_SHRINK);
        }
    } else {
        for (SwipeLinearLayout sll : swipeLinearLayouts) {
            if (null == sll) {
                continue;
            }
            if (!sll.equals(thisSll)) {
                //划开一个sll， 其他收缩
                sll.scrollAuto(SwipeLinearLayout.DIRECTION_SHRINK);
            }
        }
    }
}
```
swipeLinearLayouts是Adapter中定义的，一个保存ListView中所有item里的SwipeLinearLayout的列表(由于convertView的复用，其实这个列表的长度是很有限的，不用担心内存等问题)。
看了代码， 实现的逻辑就很清楚了：
竖直方向，直接缩起所有SwipeLinearLayout， 否则，把不是当前滑动的SwipeLinearLayout全部缩起来。

## 总结
如果只是考虑横向滚动，那么问题就非常简单，只需要重写OnTouchEvent，这点大家肯定都会，我也没必要写这篇博客了。然而为了处理滑动冲突(包括保证子View的点击)，我们将dispatchTouchEvent和onInterceptTouchEvent也都重写了。一个是保证自己在滑动的时候，事件不会被上层粗暴拦截，另一个是保证自己在不滑动的时候，事件能够传给内部的子控件。

代码只贴了重点部分， 但其实也差不多了，毕竟代码量也不是很大，重点就在于对于事件的分发与拦截。如果需要查看项目及demo完整代码，可以访问：
[android-SwipeLinearLayout](https://github.com/lankton/android-SwipeLinearLayout)


