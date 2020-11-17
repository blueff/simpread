\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[blog.csdn.net\](https://blog.csdn.net/xxzhangx/article/details/53558654)

这次来介绍下 python 第三方包的安装，前面介绍了 3 种方式，我们这次再介绍一种方式。前两种方式见下面的链接。

pip install
-----------

[http://blog.csdn.net/xxzhangx/article/details/52740227](http://blog.csdn.net/xxzhangx/article/details/52740227)

easy\_install 与 pip.exe install
-------------------------------

[http://blog.csdn.net/xxzhangx/article/details/52337349](http://blog.csdn.net/xxzhangx/article/details/52337349)

这次要说的是用 setup.py 来安装第三方包。步骤如下：

### 步骤：setup.py

*   先下载你要安装的包，并解压到磁盘下；
*   进入到该文件的 setup.py 目录下 ，打开 cmd，并切换到该目录下；
*   先执行 python setup.py build
*   然后执行 python setup.py install

回过头去官网看，下载方式都有说明，长久以来竟然没发现。泪流满面啊！[https://pypi.python.org/pypi](https://pypi.python.org/pypi)  
![](https://img-blog.csdn.net/20161210151322002?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQveHh6aGFuZ3g=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

哪种方式好用呢？
--------

个人偏爱 pip install + packages ，这种方式感觉更好，安装成功率高！为何？原因不知，感觉好用就行吧，是吧！