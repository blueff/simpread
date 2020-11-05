\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[blog.csdn.net\](https://blog.csdn.net/wishchin/article/details/78560725)

一种操作方法：

       文章链接：[MAC 下使用 Pycharm，debug 调试时怎样带参数](http://blog.csdn.net/sxd125/article/details/53138797)      

今天在网上找了一个例子敲代码，因为我使用的是 PyCharm，例子运行时需要带参数，开始不知道怎么带参数，网上搜了大半天，最终才找到自己想要的方法，记录一下。

代码中有需要使用到参数，如下图：

![](https://img-blog.csdn.net/20161112112341423)  

因为开始不知道怎么带参数，直接运行时，报错，因为没参数

![](https://img-blog.csdn.net/20161112112410650)  

运行时，至少需要一个文件参数，添加参数

在 PyCharm 中选择'Run'->Edit Configurations，如下图

![](https://img-blog.csdn.net/20161112112812214)  

在 scrip parameters 中输入参数即可，如下图，输入地址

![](https://img-blog.csdn.net/20161112113051340)

输入文件地址

![](https://img-blog.csdn.net/20161112113058667)  

注意事项：

       调试参数输入，去掉脚本文件的前两个参数。