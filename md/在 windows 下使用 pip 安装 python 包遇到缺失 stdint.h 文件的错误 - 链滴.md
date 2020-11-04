\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[ld246.com\](https://ld246.com/article/1548337418731)

今天在 windows 上使用 pip 安装一个 python 包 python-lzf 时遇到如下的错误：

```
fatal error C1083: Cannot open include file: 'stdint.h': No such file or directory
  error: command 'C:\\\\Users\\\\wxyuan\\\\AppData\\\\Local\\\\Programs\\\\Common\\\\Microsoft\\\\Visual C++ for Python\\\\9.0\\\\VC\\\\Bin\\\\cl.exe' failed with exit status 2
```

从错误提示可以看出，是由于缺少了 `stdint.h` 造成的错误，经过一番搜索找到了解决方法。

> (1). 从这里 [https://www.microsoft.com/download/details.aspx?id=44266](https://link.ld246.com/forward?goto=https%3A%2F%2Fwww.microsoft.com%2Fdownload%2Fdetails.aspx%3Fid%3D44266) 下载软件 **Microsoft Visual C++ Compiler for Python 2.7**(事实上该软件我已经安装了)  
> (2). 确认安装路径，比如我的安装路径是 **C:\\Users\\wxyuan\\AppData\\Local\\Programs\\Common\\Microsoft\\Visual C++ for Python\\9.0**  
> (3). 下载 `stdint.h` 文件，下载地址 [https://github.com/mattn/gntp-send/blob/master/include/msinttypes/stdint.h](https://link.ld246.com/forward?goto=https%3A%2F%2Fgithub.com%2Fmattn%2Fgntp-send%2Fblob%2Fmaster%2Finclude%2Fmsinttypes%2Fstdint.h)  
> (4). 拷贝 `stdint.h` 文件到安装路径的 **VC\\include** 目录下，比如我的路径是 **C:\\Users\\wxyuan\\AppData\\Local\\Programs\\Common\\Microsoft\\Visual C++ for Python\\9.0\\VC\\include**

最后重新安装 python-lzf 包，成功！

关于为什么安装的 **Microsoft Visual C++ Compiler for Python 2.7** 缺少了 `stdint.h` 文件，这篇文章有介绍 [http://stephendoyle.blogspot.com/2008/03/c-tr1-stdinth-still-missing-from-visual.html](https://link.ld246.com/forward?goto=http%3A%2F%2Fstephendoyle.blogspot.com%2F2008%2F03%2Fc-tr1-stdinth-still-missing-from-visual.html)

*   [Python](https://ld246.com/tag/python)
    
    [Python](https://link.ld246.com/forward?goto=https%3A%2F%2Fwww.python.org) 是一种面向对象、直译式电脑编程语言，具有近二十年的发展历史，成熟且稳定。它包含了一组完善而且容易理解的标准库，能够轻松完成很多常见的任务。它的语法简捷和清晰，尽量使用无异义的英语单词，与其它大多数程序设计语言使用大括号不一样，它使用缩进来定义语句块。
    
    449 引用 • 623 回帖 • 525 关注
    

726 13 351 41 84 44 3 22 145