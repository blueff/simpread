> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/jiulousanti/article/details/23343687) **一、原理篇**

       ccache （“compiler cache” 的缩写）工具通过将头文件高速缓存到源文件之中而改进了构建性能，因而通过减少每一步编译时添加头文件所需要的时间而提高了构建速度。  
       在标准的编译过程中，在 UNIX 下使用 C/C++ 开发应用程序通常需要用到一个编译器（如 gcc）以及一个编译工具，比如 make。make 和所有的 C 编译器的问题在于 C 预处理程序（preprocessor）和头文件的工作方式。观察一个典型的 C 源文件，您会发现其中有很多由 #include 所引用的各种头文件。  
       每一次编译一个文件时，C 预处理程序（cpp）都会解析并引入每个头文件以及这些头文件引用到的任何文件。通过对内容进行解析，ccp 可以将一个相当基本的 1-KB 大小的源文件转化为一个 8-KB 大小的源文件，在这个过程中，会合并入几十个甚至几百个头文件。在典型的开发项目中，有很多与项目相关的头文件可能会在不同的源文件中多次被引入，而且每个头文件本身也可能引用很多其他头文件。  
      在典型的编译过程中，make 工具只编译自上次编译后发生修改的文件，这样就在很大程度上简化了编译过程。make 将必须被编译的文件限制在经过修改的那些源文件范围之内，但是即使是使用 make，仍然有相当可观的浪费。每一次编译项目时，源文件在编译为汇编语言和最终的机器代码之前，都要通过 cpp 进行解析。对每一个文件来说，每一次可能都要重新解析头文件。从编译的全过程来看，您最后可能多次解析了相同的头文件，浪费了处理器周期，更重要的是浪费了开发者的时间，因为他们要等待这一过程的完成。在一个团队中，这一影响可能会更为明显，因为多名开发者可能都会反复多次重复这一过程，在某一天甚至可能会同时进行。  
       ccache 工具会高速缓存编译生成的信息，并在编译的特定部分使用高速缓存的信息，比如头文件，这样就节省了通常使用 cpp 解析这些信息所需要的时间。如果您编译清单 2 中的文件，假定 foobar.h 中包含对其他头文件的引用，ccache 会用那个文件的 cpp-parsed 版本来取代 include 声明。就那么简单。不是真正去读取、理解并解释其内容，ccache 只是将最终的文本拷贝到文件中，使得它可以立即被编译。  
      清单 2. 源文件内容  
       #include "foobar.h"  

       void main

**二、实践篇**

**1、开启 android ccache**  
(1). ccache 是 google 默认就存放在 codebase 的工具（在 prebuilts/misc/linux-x86/ccache 目录下），对于 ccache 的功能网上有较多资料。  
(2). 设置 cache 大小（这一步一定要做！！），一个 codebase 需要大约 5G 的空间，如果你的编译环境有 5 个 codebase 在使用，那么需要 25G 的空间。设置小了还会引起编译时间延长！！  
     [1].GB 版本：prebuilt/linux-x86/ccache/ccache -M 25G  
     [2].JB 版本：prebuilts/misc/linux-x86/ccache/ccache -M 25G  
     [3] Kk 版本：prebuilts/misc/linux-x86/ccache/ccache -M 25G  
      * 该设置会永久存储在 CCACHE_DIR，  
(3). 开启的方法：  
    [1]. 命令行增加 USE_CCACHE=1 即可：./mk -o=USE_CCACHE=1 n。如果是 user 版本开启 ccahce 的话，可以这样：./mk -o=USE_CCACHE=1,TARGET_BUILD_VARIANT=user n。  
    [2]. 建议: 添加到环境变量 (其他方式亦可)：修改 alps/makeMtk，在 #!/usr/bin/perl 之后新建一行，填写 $ENV{"USE_CCACHE"} = 1;  
(4). 第 1 次编译会花多一点时间建立 cache，之后即可享受 ccache 带来的好处，降低所有使用 gcc 的编译时间，大约 25%~30%。  
    [1]. 可以通过 ccache -s 查看 cache 信息，如果满了可以用 ccache -C 清除所有。在满了状态下编译新 codebase，将会旧的那个清除，这会降低效率，所以 cache 一定要设置正确！  
(6). 对 mm/new/remake android 都有效果。  
(7) 可以使用以下命令来查看 ccache 的运行状况：  

    $ watch -n1 -d prebuilt/linux-x86/ccache/ccache -s   （工程目录下运行）

**2、具体操作：**

1、配环境变量：

     a. 进入 usr 目录打开. bashrc 文件

     b. 在. bashrc 文件中增加如下信息并保存：

              export USE_CCACHE=1

              export CCACHE_DIR=~/.ccache

2、配置 ccache 大小

      a. 进入 Android 工程目录 [kk 版本，红色为 Android 工程中的路径]：

             /home/hwruanjian/WorkArea/android_mtk_kk1_6592_mp/android_mtk_kk1_6592_mp/prebuilts/misc/linux-x86/ccache

      b. 这里选择配 30G，在上面的目录下执行以下命令：

           ./ccache -M 30G

3、正常编译 Android 工程

      ./make xxxxx new