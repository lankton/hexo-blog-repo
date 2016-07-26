---
title: 【Android】在onCreate中直接进行Activity跳转
date: 2016-07-26 20:45:57
categories: Lan's tech
tags:
  - Android
---
## 前言
关于Activity的生命周期，相信所有的Android developer都已经非常熟悉，这里不再赘述。这次做了一个试验，验证了一下在A活动的onCreate中直接启动B活动，2个Activity的生命周期会是一个怎样的调用过程。

## 代码
### MainActivity
```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Log.v("lanktondebug", "main oncreate");
        Intent intent = new Intent(this, SubActivity.class);
        this.startActivity(intent);
        Log.v("lanktondebug", "main oncreate after jump");
    }
    

    @Override
    protected void onStart() {
        super.onStart();
        Log.v("lanktondebug", "main onstart");

    }

    @Override
    protected void onResume() {
        super.onResume();
        Log.v("lanktondebug", "main onresume");
    }

    @Override
    protected void onPause() {
        super.onPause();
        Log.v("lanktondebug", "main onpause");
    }

    @Override
    protected void onStop() {
        super.onStop();
        Log.v("lanktondebug", "main onstop");
    }
}
```
### SubActivity
与上面类似， 将Log中的main 换成 sub， 同时onCreate里略有改动

````java 
@Override  
protected void onCreate(Bundle savedInstanceState) {
    Log.v("lanktondebug", "sub oncreate");
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
}
````

## 运行结果
```
07-26 20:41:48.816 1969-1969/? V/lanktondebug: main oncreate
07-26 20:41:48.823 1969-1969/? V/lanktondebug: main oncreate after jump
07-26 20:41:48.826 1969-1969/? V/lanktondebug: main onstart
07-26 20:41:48.826 1969-1969/? V/lanktondebug: main onresume
07-26 20:41:48.832 1969-1969/? V/lanktondebug: main onpause
07-26 20:41:48.929 1969-1969/? V/lanktondebug: sub oncreate
07-26 20:41:48.937 1969-1969/? V/lanktondebug: sub onstart
07-26 20:41:48.937 1969-1969/? V/lanktondebug: sub onresume
07-26 20:41:49.374 1969-1969/? V/lanktondebug: main onstop
```

试验做了好几次， 都是这样的log顺序。 具体不分析了， Log已经很清楚了。