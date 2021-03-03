> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/weixin_39947522/article/details/110475013)

最近由于实际需要，开始使用 VScode 来写 Python。由于 VScode 并不像 Pycharm 是为 Python 定制的编译工具，要想舒舒服服的在 VScode 上写代码，一些配置是必不可少的，废话不多说，上干货：

首先说一句，若插件无法成功安装，请移步 VScode 官网，下载最新版本

一、Python

必必必必必须安装的插件，官方出品，值得拥有。

![](https://img-blog.csdnimg.cn/img_convert/511ab6fc4cd302098953c5ae57dcdce7.png)

安装了就完事了？当然不，需要修改两个地方才能有用核心功能：code navigation（代码跳转） 和 auto-completion（自动补全）

首先进入 Setting 目录：File->Preferences->Settings

1）、code navigation：

![](https://img-blog.csdnimg.cn/img_convert/02d40d84575f83594ec8af899b9eaf46.png)

2）、auto-completion:

![](https://img-blog.csdnimg.cn/img_convert/d760e0539a60506c1f66e2a504014b2a.png)

修改后重启 Vscode，即可生效。

二、Remote-SSH

这个是官方推出的远程编程神器。有需要远程如：Linux 编程的小伙伴注意了。

![](https://img-blog.csdnimg.cn/img_convert/91b765401e4fe9890b59c96047db4523.png)

配置方法在这里：

VSCode Remote 体验 | 远程 Linux 环境开发真香​www.jianshu.com

![](https://img-blog.csdnimg.cn/img_convert/3ee4d44294fea52329e1d60b18d459d6.png)

实用的插件，这两个足够了，还有很多个性化的插件大家可以自行探索。

三、VScode + Pipenv：

当前 Pipenv 可是非常的火爆，但是如此一来 VScode 就成了盲人，完全找不到 Pipenv 中的 Packages，找不到 Packages 的代码，全文泛红，无法调试。不要着急，只需如下操作即可解决：

1）、首先使用指令获取 pipenv 存放虚拟环境的文件夹目录：

pipenv --venv

结果如：/home/**username**/.local/share/virtualenvs/**project-xxxx**

**注：**若提示未激活 pipenv 环境，使用命令：pipenv shell 激活即可。

2）、再次进入 VScode 的 Setting 目录：File->Preferences->Settings，修改如下两处：

![](https://img-blog.csdnimg.cn/img_convert/a5f18ff1fba9bfcdac1e1d1fecbdbce5.png)

![](https://img-blog.csdnimg.cn/img_convert/6702547d384e24570289b9bd7eaf39c1.png)

修改后无需重启 VScode，即时生效。

基本上就这么多了，希望对大家有所帮助，少走弯路。