> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/dbe579f6cc84)

一、反编译未加固的 apk
=============

### 1、反编译用到的工具

[apktool](https://links.jianshu.com/go?to=https%3A%2F%2Fbitbucket.org%2FiBotPeaches%2Fapktool%2Fdownloads)：资源文件获取，可以提取出图片文件和布局文件进行使用查看  
[dex2jar](https://links.jianshu.com/go?to=%255Bhttp%3A%2F%2Fsourceforge.net%2Fprojects%2Fdex2jar%2Ffiles%2F%255D%28http%3A%2F%2Fsourceforge.net%2Fprojects%2Fdex2jar%2Ffiles%2F%29)：将 apk 反编译成 java 源码（classes.dex 转化成 jar 文件）  
[jd-gui](https://links.jianshu.com/go?to=http%3A%2F%2Fjd.benow.ca%2F)：查看 APK 中 classes.dex 转化成出的 jar 文件，即源码文件

### 2、使用 apktool 反编译资源文件和 Manifest 文件

为了方便操作，首先将下载好的 3 个工具统一放到 1 个文件夹中

![](http://upload-images.jianshu.io/upload_images/2595400-5ff245d463d4457a.png) image.png

将 gui 和 dex2jar 分别解压到当前文件夹

![](http://upload-images.jianshu.io/upload_images/2595400-ad658d5761024e08.png) image.png

使用 apktool 反编译 apk 得到图片、xml、maifest 等文件  
运行 CMD.exe, 进入 Demo.apk 路径下  
运行 apktool 对 apk 进行反编译  
使用如下命令将 Demo.apk 反编译到【Demo】文件夹下：

```
java -jar apktool_2.4.0.jar d -f Demo.apk -o Demo
```

打开 Demo 文件，就可以看到反编译后生成的文件

![](http://upload-images.jianshu.io/upload_images/2595400-1280413d68abb16b.png) image.png

这时候可以看到【res】和 Manifest 文件了；

### 3、使用 dex2jar 反编译 java 源码

将 Demo.apk 直接修改后缀，改为 Demo.zip，并解压  
解压后的文件夹中，classes.dex 就是 java 源码打包后的文件

![](http://upload-images.jianshu.io/upload_images/2595400-ce4ebbafe2b61ac5.png) image.png

将命令行定位到 dex2jar.bat 所在目录  
使用命令行：

```
d2j-dex2jar classes.dex
```

会生成 classes-dex2jar 文件，该文件就是反编译后的 java 源码文件

![](http://upload-images.jianshu.io/upload_images/2595400-6cbc08f1b3d85920.png) image.png

### 4、使用 gui 查看反编译后的 java 源码

双击 jd-gui.exe, 运行 gui，

![](http://upload-images.jianshu.io/upload_images/2595400-1fabbac3a22c8153.png) image.png

选择刚刚生成的 classes-dex2jar，打开后就可以看到源码（图中的 Demo 为了容易阅读没有加混淆，正常上线的应用会加混淆来加大阅读难度）

![](http://upload-images.jianshu.io/upload_images/2595400-a73080438062bbc5.png) image.png

至此，一个没有加壳的 apk 已经被反编译完成了，但是实际工作中，上线的 apk 包大多数都会进行加固，有些应用市场甚至后要求加固，例如 360，应用宝等，下面会介绍一中常见的脱壳技术

二、脱壳
====

加固后的 apk 在通过以上的反编译流程后，无法得到正常的 java 源码，如下图

![](http://upload-images.jianshu.io/upload_images/2595400-f2e8600b7dd973b0.png) 未加固. png

![](http://upload-images.jianshu.io/upload_images/2595400-03ae62c9bab06a0a.png) 360 加固保. png

![](http://upload-images.jianshu.io/upload_images/2595400-aa9337a22d6b4fe2.png) 乐加固. png

从上图可以看出，经过加固后的 apk，通过常规方法反编译无法获取到源码。

### 1、脱壳用到的工具

[FDex2](https://links.jianshu.com/go?to=https%3A%2F%2Fbbs.pediy.com%2Fthread-224105.htm) ：通过 Hook ClassLoader 的 loadClass 方法，反射调用 getDex 方法取得 Dex(com.android.dex.Dex 类对象)，再将里面的 dex 写出  
[VirtualXposed](https://links.jianshu.com/go?to=https%3A%2F%2Fvxposed.com%2F)：无需 root 手机即可使用的 xp 框架

### 2、脱壳

Step1、将 【VirtualXposed】、【FDex2】和需要脱壳的 apk 都安装到手机上  
Step2、启动【VirtualXposed】，并在【VirtualXposed】中安装【FDex2】：

![](http://upload-images.jianshu.io/upload_images/2399767-71477861782392dd.gif) image

Step3、在【VirtualXposed】中激活【FDex2】:

![](http://upload-images.jianshu.io/upload_images/2399767-ba92e116ce7bc010.gif) image

Step4、在【VirtualXposed】中安装要脱壳的应用，方法和 Step2 一样。

Step5: 启动【VirtualXposed】中的 F【Dex2】，并配置要脱壳的应用。

![](http://upload-images.jianshu.io/upload_images/2595400-ea1b8d3b417f3f40.png) image.png

Step6：在【VirtualXposed】中运行要脱壳的应用。  
Step7：在上【VirtualXposed】安装文件管理器，方便将生成的 dex 文件导出  
在【VirtualXposed】选中【高级设置】

![](http://upload-images.jianshu.io/upload_images/2595400-75a010447f24481b.png) image.png

选中【文件管理】，会提示安装【Amaze】文件管理器

![](http://upload-images.jianshu.io/upload_images/2595400-a728419bda6313e2.png) image.png

Step8：查看脱壳后的 dex 文件，打开【Amaze】, 选中【Private Storage】，定位到 FDex2 设置的 dex 输出路径下，可以看到脱壳后的 dex 文件

![](http://upload-images.jianshu.io/upload_images/2595400-28abe19bed9eca3b.png) image.png

![](http://upload-images.jianshu.io/upload_images/2595400-06d4857c9544465c.png) image.png

Step9：将 dex 文件导出：长按 dex 文件，点击顶部的复制按钮，然后回到选中【External Storage】并选择一个路径，点击顶部的粘贴按钮，将文件粘贴到外部存储，这时，在手机自带的文件管理器中就可以找到刚刚的 dex 文件了

![](http://upload-images.jianshu.io/upload_images/2595400-ed3133ac6994121c.png) image.png

![](http://upload-images.jianshu.io/upload_images/2595400-9db146386d8ea4e1.png) image.png

![](http://upload-images.jianshu.io/upload_images/2595400-ef0b358ccabcc98f.png) image.png

Step10：通过 dex2jar 对脱壳的 dex 文件进行反编译，得到的 3 个 dex 文件中，有的是无用的文件，可以依次打开看看，最后可以看到，脱壳后的 dex 文件被成功反编译后

![](http://upload-images.jianshu.io/upload_images/2595400-46cd78985499bf19.png) image.png

至此，脱壳完成

源代码和相关工具类已上传码云  
[https://gitee.com/xiaobug/XpDemo](https://links.jianshu.com/go?to=https%3A%2F%2Fgitee.com%2Fxiaobug%2FXpDemo)  
[https://gitee.com/xiaobug/reverse_tool](https://links.jianshu.com/go?to=https%3A%2F%2Fgitee.com%2Fxiaobug%2Freverse_tool)

参考链接，感谢作者的无私奉献  
1、[APK 反编译](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fs13383754499%2Farticle%2Fdetails%2F78914592)  
2、[Android APK 脱壳 -- 腾讯乐固、360 加固一键脱壳](https://www.jianshu.com/p/138c9de2c987)  
3、[【Xposed 模块开发入门】](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fmrglaucusss%2Farticle%2Fdetails%2F50963542)  
4、[Xposed 热更新（干货，建议收藏）](https://www.jianshu.com/p/360edca021bb)  
5、[android apk 编译打包过程](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fbrycegao321%2Farticle%2Fdetails%2F79127159)  
6、[Android 工程师，如何简单高效的学会 smali 语法](https://www.jianshu.com/p/b23782460f61)  
7、[使用 baksmali 及 smali 修改 apk 并打包](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2FMoonDroid%2Farticle%2Fdetails%2F44224111)  
8、[MT 管理器](https://links.jianshu.com/go?to=http%3A%2F%2Fbinmt.cc%2Fdoc%2Findex.php%3Fs%3D%2F1%26page_id%3D1)  
9、[Oo0 代码混淆实现方法](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FT8CFMjQ5_te1fIHzwzTpUg)  
10、[修改 AndroidStudio 混淆配置文件](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fu014476720%2Farticle%2Fdetails%2F85112374)