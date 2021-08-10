> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/QI_Rainbow/article/details/78114752)

**做 Android 系统开发时，有时会需要自定义物理按键的功能，我们首先要知道每个按键对应的键值，然后根据需求更改每个键的功能。**  

1. 按下物理按键时获取到十六进制值（例如：d4），通过 adb 查看：  

```
adb shell 
getevent /dev/input/event1
```

2. 将十六进制值转化成十进制后，在 Generic.kl 中查看驱动层对应的键值，也就是上报的 keycode 值（例如：d4 转化成十进制为 212, 对应 keycode 为 CAMERA），Generic.kl 文件存放位置：

```
case KeyEvent.KEYCODE_CAMERA:{
 
     if(down) {
 
        mContext.sendBroadcast(new Intent("android.intent.action.CAMERA_BTN_CLICK"));
 
     }
 
    break;
 
}
```

查看 Generic.kl 文件中 212 对应的 keycode：

```
key 212 CAMERA
```

3. 将上报的 keycode 值定义为 integer：

frameworks\base\core\java\android\view\KeyEvent.java

```
public static final int KEYCODE_CAMERA = 27;
```

4. 处理 keyevent 的 keycode 值（例如：KEYCODE_CAMERA 或 27）：

frameworks\base\policy\src\com\android\internal\policy\impl\PhoneWindowManager.java  

例如：  

```
case KeyEvent.KEYCODE_CAMERA:{
 
     if(down) {
 
        mContext.sendBroadcast(new Intent("android.intent.action.CAMERA_BTN_CLICK"));
 
     }
 
    break;
 
}
```