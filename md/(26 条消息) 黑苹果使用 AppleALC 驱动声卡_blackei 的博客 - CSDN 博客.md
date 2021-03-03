> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/blackei/article/details/88215718)

我的是台式机，安装的系统版本 10.13.6（17G65），使用的是带 clover 的镜像，一些驱动已经被大牛弄好了，我只需要弄好显卡驱动和声卡驱动就搞定驱动了，这里记录一下 AppleALC 使用的教程。

AppleALC.kext 和 Lilu.kext 文件一般已经在带 clover 的镜像里了，安装好后去 efi 里找到就行了，放在 other 文件夹里就行了，有时候会被放到 backup 文件夹里了，拖出来就行了。也可以去 github 上下载最新版。

地址为 [https://github.com/acidanthera/AppleALC](https://github.com/acidanthera/AppleALC)

Other 路径：EFI/CLOVER/kexts/Other；backup 就在 other 文件夹下。

当 AppleALC.kext 和 Lilu.kext 都放到 other 文件夹下后使用 clover configurator 打开 config.plist 修改 layout_id，我的声卡是 alc887，所以 id 可以填 99，也许你的不是 99，你可以 1~99 多试几个数值。每个声卡的数值范围可以在 github 上找到。这里有声卡列表，找到自己的声卡就可以查看支持哪些 id 值：

[https://github.com/acidanthera/AppleALC/tree/master/Resources](https://github.com/acidanthera/AppleALC/tree/master/Resources)

![](https://img-blog.csdnimg.cn/20190306005334896.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JsYWNrZWk=,size_16,color_FFFFFF,t_70)

修改后保存，然后重启。

下载和使用 clover configurator：[https://blog.csdn.net/blackei/article/details/88216473](https://blog.csdn.net/blackei/article/details/88216473)