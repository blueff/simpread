\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/9bcd6671c00e)

Audiolab 是一个使用 numpy 数组的音频文件 IO 的 python 包。它支持许多不同的音频格式，包括 wav，aiff，au，flac，ogg，htk。它还支持输出到音频设备（仅限 Mac OS X 和 Linux）。  
为简单起见，提供了类似 matlab 的 API，用于简单的导入 / 导出; 还提供更完整的 API。

pip install scikits.audiolab  
提示报错

```
\[root@localhost audiolab\]# pip install scikits.audiolab -i http://pypi.douban.com/simple --trusted-host pypi.douban.com
Looking in indexes: http://pypi.douban.com/simple
Collecting scikits.audiolab
  Downloading http://pypi.doubanio.com/packages/b0/d8/d9babf3e4fa3ac8094e1783415bf60015a696779f4da4c70ae6141aa5e3a/scikits.audiolab-0.11.0.tar.gz (892kB)
     |████████████████████████████████| 901kB 1.3MB/s 
    ERROR: Command errored out with exit status 1:
     command: /root/anaconda3/envs/speechvalley/bin/python -c 'import sys, setuptools, tokenize; sys.argv\[0\] = '"'"'/tmp/pip-install-kuux06xf/scikits.audiolab/setup.py'"'"'; \_\_file\_\_='"'"'/tmp/pip-install-kuux06xf/scikits.audiolab/setup.py'"'"';f=getattr(tokenize, '"'"'open'"'"', open)(\_\_file\_\_);code=f.read().replace('"'"'\\r\\n'"'"', '"'"'\\n'"'"');f.close();exec(compile(code, \_\_file\_\_, '"'"'exec'"'"'))' egg\_info --egg-base pip-egg-info
         cwd: /tmp/pip-install-kuux06xf/scikits.audiolab/
    Complete output (49 lines):
    SndfileInfo:
    customize UnixCCompiler
      libraries sndfile not found in /root/anaconda3/envs/speechvalley/lib
    customize UnixCCompiler
      libraries sndfile not found in /usr/local/lib64
    customize UnixCCompiler
      libraries sndfile not found in /usr/local/lib
    customize UnixCCompiler
      libraries sndfile not found in /usr/lib64
    customize UnixCCompiler
      libraries sndfile not found in /usr/lib
    customize UnixCCompiler
      libraries sndfile not found in /usr/lib/
    Traceback (most recent call last):
      File "scikits/audiolab/pysndfile/setup.py", line 20, in configuration
        sf\_config = sf\_info.get\_info(2)
      File "/root/anaconda3/envs/speechvalley/lib/python3.5/site-packages/numpy/distutils/system\_info.py", line 645, in get\_info
        raise self.notfounderror(self.notfounderror.\_\_doc\_\_)
    numpy.distutils.system\_info.NotFoundError: Some third-party program or library is not found.
    
    During handling of the above exception, another exception occurred:
    
    Traceback (most recent call last):
      File "<string>", line 1, in <module>
      File "/tmp/pip-install-kuux06xf/scikits.audiolab/setup.py", line 85, in <module>
        classifiers=CLASSIFIERS)
      File "/root/anaconda3/envs/speechvalley/lib/python3.5/site-packages/numpy/distutils/core.py", line 137, in setup
        config = configuration()
      File "/tmp/pip-install-kuux06xf/scikits.audiolab/setup.py", line 71, in configuration
        config.add\_subpackage(DISTNAME)
      File "/root/anaconda3/envs/speechvalley/lib/python3.5/site-packages/numpy/distutils/misc\_util.py", line 1036, in add\_subpackage
        caller\_level = 2)
      File "/root/anaconda3/envs/speechvalley/lib/python3.5/site-packages/numpy/distutils/misc\_util.py", line 1005, in get\_subpackage
        caller\_level = caller\_level + 1)
      File "/root/anaconda3/envs/speechvalley/lib/python3.5/site-packages/numpy/distutils/misc\_util.py", line 942, in \_get\_configuration\_from\_setup\_py
        config = setup\_module.configuration(\*args)
      File "scikits/audiolab/setup.py", line 10, in configuration
        confgr.add\_subpackage('pysndfile')
      File "/root/anaconda3/envs/speechvalley/lib/python3.5/site-packages/numpy/distutils/misc\_util.py", line 1036, in add\_subpackage
        caller\_level = 2)
      File "/root/anaconda3/envs/speechvalley/lib/python3.5/site-packages/numpy/distutils/misc\_util.py", line 1005, in get\_subpackage
        caller\_level = caller\_level + 1)
      File "/root/anaconda3/envs/speechvalley/lib/python3.5/site-packages/numpy/distutils/misc\_util.py", line 942, in \_get\_configuration\_from\_setup\_py
        config = setup\_module.configuration(\*args)
      File "scikits/audiolab/pysndfile/setup.py", line 25, in configuration
        site.cfg file, in section \[sndfile\].""")
    numpy.distutils.system\_info.NotFoundError: sndfile (http://www.mega-nerd.com/libsndfile/) library not found.
    Directories to search for the libraries can be specified in the
    site.cfg file, in section \[sndfile\].
    ----------------------------------------
ERROR: Command errored out with exit status 1: python setup.py egg\_info Check the logs for full command output.
```

提示需要安装 sndfile。

下载 audiolab：

```
\# wget https://files.pythonhosted.org/packages/b0/d8/d9babf3e4fa3ac8094e1783415bf60015a696779f4da4c70ae6141aa5e3a/scikits.audiolab-0.11.0.tar.gz
```

解压后通过 python3 setup.py install 安装出错，也是提示没有找到 sndfile

下载 libsndfile

```
#wget http://www.mega-nerd.com/libsndfile/files/libsndfile-1.0.28.tar.gz
```

解压并安装：

```
\# tar xzvf libsndfile-1.0.28.tar.gz
# cd libsndfile-1.0.28
# .configure
# make
# make install
```

centos 下的 site.cfg 文件：

```
\[root@localhost audiolab\]# cat site.cfg
\[sndfile\]
include\_dirs = /usr/local/include
library\_dirs = /usr/local/lib/
sndfile\_libs = sndfile
```

Ubuntu 下的 site.cfg 文件：

```
\[sndfile\]
include\_dirs = /usr/include
library\_dirs = /usr/lib/x86\_64-linux-gnu/
sndfile\_libs = sndfile
```

在 audiolab 目录下，再次执行 python setup.py install 可以了。

audiolab 官网：  
[http://cournape.github.io/audiolab/](https://links.jianshu.com/go?to=http%3A%2F%2Fcournape.github.io%2Faudiolab%2F)

libsndfile 官网：  
[http://www.mega-nerd.com/libsndfile](https://links.jianshu.com/go?to=http%3A%2F%2Fwww.mega-nerd.com%2Flibsndfile)