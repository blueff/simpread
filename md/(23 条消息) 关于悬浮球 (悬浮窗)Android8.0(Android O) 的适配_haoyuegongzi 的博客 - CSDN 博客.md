> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/haoyuegongzi/article/details/81699699)

最近项目新需求要用到悬浮球技术，因以前的工作项目中就用到了这个技术，因此看到这个需求时心里是愉悦的，心念着照搬过来即可。开发中也确实如此，直接搬过来，然后测试了三星 C5，小米 Note4 以及 OPPO 手机，运行顺畅，均无异常。当测试到华为 Honor8 青春版时，就 GG 了。抛出了以下异常：

```
java.lang.RuntimeException: Unable to create service com.haoyue.notedemos.floatball.FloatServive: android.view.WindowManager$BadTokenException: 

Unable to add window android.view.ViewRootImpl$W@270feab -- 
******permission denied for window type 2002******

    at android.app.ActivityThread.handleCreateService(ActivityThread.java:3878)
    at android.app.ActivityThread.-wrap5(Unknown Source:0)
    at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1989)
    at android.os.Handler.dispatchMessage(Handler.java:108)
    at android.os.Looper.loop(Looper.java:166)
    at android.app.ActivityThread.main(ActivityThread.java:7425)
    at java.lang.reflect.Method.invoke(Native Method)
    at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:245)
    at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:921)
 Caused by: android.view.WindowManager$BadTokenException: 

******Unable to add window android.view.ViewRootImpl$W@270feab -- ******
******permission denied for window type 2002******

    at android.view.ViewRootImpl.setView(ViewRootImpl.java:908)
    at android.view.WindowManagerGlobal.addView(WindowManagerGlobal.java:372)
    at android.view.WindowManagerImpl.addView(WindowManagerImpl.java:128)
    at com.haoyue.notedemos.floatball.FloatServive.initFloatBall(FloatServive.java:64)
    at com.haoyue.notedemos.floatball.FloatServive.onCreate(FloatServive.java:34)
    at android.app.ActivityThread.handleCreateService(ActivityThread.java:3858)
    at android.app.ActivityThread.-wrap5(Unknown Source:0) 
    at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1989) 
    at android.os.Handler.dispatchMessage(Handler.java:108) 
    at android.os.Looper.loop(Looper.java:166) 
    at android.app.ActivityThread.main(ActivityThread.java:7425) 
    at java.lang.reflect.Method.invoke(Native Method) 
    at com.android.internal.os.Zygote$MethodAndArgsCaller.run(Zygote.java:245) 
    at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:921)
```

异常的重点在这里：**Unable to add window android.view.ViewRootImpl$W@270feab –permission denied for window type 2002**，大体上说就是需要 Window type 的权限。这就真 GGJJ 了，不是给了：  
**mLayoutParams.type = WindowManager.LayoutParams.TYPE_PHONE** 的权限么？

工程中关于该部分的代码如下：

```
mLayoutParams = new WindowManager.LayoutParams();
    mManager = (WindowManager) getApplication().getSystemService(WINDOW_SERVICE);
    mLayout = (LinearLayout) LayoutInflater.from(getApplication()).inflate(R.layout.flaot_ball, null);
    mLayoutParams.type = WindowManager.LayoutParams.TYPE_PHONE;
    mLayoutParams.format = PixelFormat.RGBA_8888;
    mLayoutParams.flags = WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE;
    mLayoutParams.gravity = Gravity.LEFT | Gravity.TOP;
    mLayoutParams.x = 100;
    mLayoutParams.y = 100;
    mLayoutParams.width = WindowManager.LayoutParams.WRAP_CONTENT;
    mLayoutParams.height = WindowManager.LayoutParams.WRAP_CONTENT;
    mManager.addView(mLayout, mLayoutParams);
```

xml 文件清单也配置了这两条权限：

```
<uses-permission android:/>
<uses-permission android: />
```

查询了很多资料，没几篇文章能够说清楚怎么回事，怎么解决。好吧，回到原点，排查该异常的原因。查来查去，除了发现华为 honor 青春版时 8.0 版本的系统外，就没有别的特别明显的不同了。难道是 8.0 版本的问题？

再次带着问题查询资料，不同的是目的性更强：Android8.0 下的悬浮球。终于，找到了原因，有[大神](https://blog.csdn.net/AmazonUnicon/article/details/80986035)遇到相同的问题并解释清了抛该异常的原因：

在 Android O 之前的系统中申请了该权限后，再给对应的 window 设置：

```
mLayoutParams.type = WindowManager.LayoutParams.TYPE_PHONE即可。
```

但是在 Android O 的系统中，除了在 xml 文件清单中配置权限外，google 规定还要申请权限：.  
mLayoutParams.type = WindowManager.LayoutParams.**_TYPE_APPLICATION_OVERLAY_**;  
只有这样，悬浮窗才能显示出来。

所以，在授予权限时，要对手机的版本做如下判断：

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    mLayoutParams.type = WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY;
}else {
    mLayoutParams.type = WindowManager.LayoutParams.TYPE_PHONE;
}
```

SDK 版本在 Android8.0（Android 0）以上，授予 **TYPE_APPLICATION_OVERLAY** 权限，在这之下的，授予 TYPE_PHONE 权限。不能为了省事，统一的授予 **TYPE_APPLICATION_OVERLAY**，否则 8.0 之下的机型还是会继续报前面的错误。