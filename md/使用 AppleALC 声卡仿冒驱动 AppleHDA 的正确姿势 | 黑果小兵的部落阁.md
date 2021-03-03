> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.daliansky.net](https://blog.daliansky.net/Use-AppleALC-sound-card-to-drive-the-correct-posture-of-AppleHDA.html)

经常给网友仿冒声卡驱动, 一直都没有将过程写下来, 鉴于每天会有许多人问如何仿冒声卡, 就想将过程记录下来, 供大家参考学习

[](#通过clover提取codec)通过`CLOVER`提取`codec`
---------------------------------------

操作方法：

*   `CLOVER` v2.4k r4886 或者以上，[CLOVER 下载链接](https://github.com/Dids/clover-builder/releases)
*   在安装`CLOVER`时请勾选`AudioDex-64.efi`
*   `CLOVER`引导界面，按`F8`快捷键，它会在`EFI/CLOVER/misc`目录下生成以`HdaCodec`开头的文件，其中的`HdaCodec#x (Realtek ALCxxx).txt`就是你需要提取的`codec`

[](#通过linux提取codec)通过 linux 提取 codec
------------------------------------

### [](#制作ubuntu-linux启动u盘)制作 Ubuntu Linux 启动 U 盘:

*   下载 Ubuntu Linux 镜像, 可以选择国内开源镜像站点下载:  
    [网易](http://mirrors.163.com/ubuntu-releases/) [搜狐](http://mirrors.sohu.com/ubuntu-releases/) [阿里云](https://mirrors.aliyun.com/ubuntu-releases/)

> 如果你只是想通过 Linux 提取 codec 的话, 可以随便下载个旧的版本即可, 比如这个 [ubuntu 14.04.5LTS](https://mirrors.aliyun.com/ubuntu-releases/14.04/ubuntu-14.04.5-desktop-amd64.iso), 如果想作为日后的生产力工具的话, 我推荐你下载最新发布的 [ubuntu 17.10](https://mirrors.aliyun.com/ubuntu-releases/17.10/ubuntu-17.10-desktop-amd64.iso)

#### [](#制作ubuntu-linux安装盘)制作 Ubuntu Linux 安装盘

*   Windows 下请使用工具`UltraISO`, 方法略
    
*   macOS 下制作过程:
    
    *   插入 U 盘, 确定设备名, 方法为打开磁盘工具, 选择 U 盘, 可以看到 U 盘下面有两个分区:`disk4s1`和`disk4s2`, 那么 U 盘的设备名就是:`disk4`[![](http://7.daliansky.net/diskutil_disk4.png)](http://7.daliansky.net/diskutil_disk4.png)
        
    *   当然最简单的方法还是直接使用命令查看:`diskutil list`, 输出的信息为:
        
    
    ```
    /dev/disk4 (internal, physical):
    
    #:                  TYPE NAME      SIZE IDENTIFIER
    0:     Apple_partition_scheme   *2.0 GB disk4
    1:        Apple_partition_map    4.1 KB disk4s1
    2:                  Apple_HFS    2.4 MB disk4s2
    ```
    
    这个`disk4`就是你要操作的设备名, 后面我们会用到它
    
    *   卸载 U 盘, 准备写入镜像, 输入命令:
        
        ```
        diskutil umountDisk disk4 # 卸载U盘
        ```
        
    *   使用`dd`命令将下载的 Ubuntu Linux 镜像恢复到 U 盘上, 操作之前我有必要**提醒各位小白, 万一你不小心输入错了设备名, 那么你连哭的机会都没有, 因为 dd 是按扇区直接物理写入磁盘, 别到时候找不到数据了再悔不当初没有认真看到这段文字**
        
    
    ```
    sudo dd if=/Users/sky/Downloads/ISO/ubuntu-17.10-desktop-amd64.iso of=/dev/disk4 bs=1m
    ```
    
    命令输入完后, 请仔细认真检查下, 尤其是`of=/dev/disk4`这里, 再三确认后回车执行, 输入用户密码后请耐心等待 6-7 分钟, 写盘速度取决于你的 U 盘, 镜像恢复的过程中不会有任何的文字输出, U 盘写入成功后会显示下面类似的输出信息:
    
    ```
    ~ % diskutil umountDisk disk4
    Unmount of all volumes on disk4 was successful
    ~ % sudo dd if=/Users/sky/Downloads/ISO/ubuntu-17.10-desktop-amd64.iso of=/dev/disk4 bs=1m
    Password:
    1431+1 records in
    1431+1 records out
    1501102080 bytes transferred in 906.101477 secs (1656660 bytes/sec)
    ```
    
    *   我的 U 盘竟然写了 906 秒 (15 分钟), 我先找个地方哭会儿去 *  
        同时系统会弹出一个警告窗口, 显示类似的信息:  
        [![](http://7.daliansky.net/dd_error.png)](http://7.daliansky.net/dd_error.png)  
        那是因为 Linux 的分区格式是 ext, 在 macOS 系统下无法识别才会报错, 但是其实一个支持 UEFI 引导的 Ubuntu Linux 17.10 启动 U 盘已经制作成功了. 点击`Ignore`忽略或者`Eject`退出 U 盘
*   现在您可以使用这个新制作的 Ubuntu Linux 安装 U 盘引导 Linux 去提取 codec
    

[](#提取codec-2)提取 codec
----------------------

开机按引导设备快捷键`F12`或者`F8`进入引导设备选单, 选择`Ubuntu Linux`所在的 USB 盘回车  
[![](http://7.daliansky.net/Boot_select.jpg)](http://7.daliansky.net/Boot_select.jpg)  
出现 Ubuntu Linux 的引导界面, 选择`Try Ubuntu without installing`, 该选项可以在不安装的情况下试用 Ubuntu  
[![](http://7.daliansky.net/Ubuntu_boot.jpg)](http://7.daliansky.net/Ubuntu_boot.jpg)  
回车后稍候会进入 Ubuntu 桌面:  
[![](http://7.daliansky.net/ubuntu1.png)](http://7.daliansky.net/ubuntu1.png)  
按组合键`CTRL+ALT+t`打开终端, 输入以下命令:

```
cd ~/Desktop/ # 进入用户桌面
cp /proc/asound/card0/codec* . # 将codec开头的文件复制到当时位置
sudo cp -R /sys/firmware/acpi/tables .  # 将acpi/tables目录复制到当时位置,tables目录包括了全部的DSDT和SSDT
ls -l   # 列表
sudo chown -R ubuntu:ubuntu *   # 将当前目录下所有文件及目录所有人修改为ubuntu
ls -l   # 列表
```

显示输出信息如下:  
[![](http://7.daliansky.net/ubuntu_codec_and_DSDT.png)](http://7.daliansky.net/ubuntu_codec_and_DSDT.png)  
将桌面上的 codec 开头的文件和 tables 目录复制到 LINUX 以外的支持写入的盘符下, Linux 下面的工作已经完成, 您可以继续试用或者重启 / 关机.

使用到的工具：`verbit.sh`, [下载链接](https://github.com/daliansky/dell7000/blob/master/hda-tools/verbit.sh)  
用法: `verbit.sh codec#0`, 它会生成如下显示的信息:

```
Verbs from Linux Codec Dump File: codec#0

Codec: Realtek ALC3246   Address: 0   DevID: 283902550 (0x10ec0256)

   Jack   Color  Description                  Node     PinDefault             Original Verbs
--------------------------------------------------------------------------------------------------------
Digital Unknown  Mic at Int N/A              18 0x12   0x90a60170   01271c70 01271d01 01271ea6 01271f90
Unknown Unknown  Line Out at Ext N/A         19 0x13   0x40000000   01371c00 01371d00 01371e00 01371f40
 Analog Unknown  Speaker at Int N/A          20 0x14   0x90170140   01471c40 01471d01 01471e17 01471f90
    1/8   Black  Speaker at Ext Rear         24 0x18   0x411111f0   01871cf0 01871d11 01871e11 01871f41
    1/8   Black  Speaker at Ext Rear         25 0x19   0x411111f0   01971c10 01971d20 01971e81 01971f02
    1/8   Black  Speaker at Ext Rear         26 0x1a   0x411111f0   01a71cf0 01a71d11 01a71e11 01a71f41
    1/8   Black  Speaker at Ext Rear         27 0x1b   0x411111f0   01b71cf0 01b71d11 01b71e11 01b71f41  
 Analog    Pink  Modem Hand at Ext N/A       29 0x1d   0x40779a2d   01d71c2d 01d71d9a 01d71e77 01d71f40
    1/8   Black  Speaker at Ext Rear         30 0x1e   0x411111f0   01e71cf0 01e71d11 01e71e11 01e71f41
    1/8   Black  HP Out at Ext Front         33 0x21   0x02211030   02171c30 02171d10 02171e21 02171f02
--------------------------------------------------------------------------------------------------------


   Jack   Color  Description                  Node     PinDefault             Modified Verbs
--------------------------------------------------------------------------------------------------------
Digital Unknown  Mic at Int N/A              18 0x12   0x90a60170   01271c70 01271d00 01271ea6 01271f90
Unknown Unknown  Line Out at Ext N/A         19 0x13   0x40000000   01371c00 01371d00 01371e00 01371f40
 Analog Unknown  Speaker at Int N/A          20 0x14   0x90170120   01471c20 01471d00 01471e17 01471f90
 Analog    Pink  Modem Hand at Ext N/A       29 0x1d   0x40779a2d   01d71c40 01d71d90 01d71e77 01d71f40
    1/8   Black  HP Out at Ext Front         33 0x21   0x02211030   02171c30 02171d10 02171e21 02171f01
--------------------------------------------------------------------------------------------------------

Modified Verbs in One Line: 01271c70 01271d00 01271ea6 01271f90 01371c00 01371d00 01371e00 01371f40 01471c20 01471d00 01471e17 01471f90 01d71c40 01d71d90 01d71e77 01d71f40 02171c30 02171d10 02171e21 02171f01
--------------------------------------------------------------------------------------------------------
```

如果你希望将输出结果保存下来, 可以将命令改为:

```
verbit.sh codec#0 > ALC256_dump.txt
```

这个 ALC256_dump.txt 就是上面显示的输出信息, 你可以使用其它的文字编辑器打开它.  
重要的信息不能漏掉了, 后面我们在编译声卡驱动的时候会经常用到它们:

```
Codec: Realtek ALC3246   Address: 0   DevID: 283902550 (0x10ec0256)
```

其中的:

<table><thead><tr><th>名称</th><th>解释</th></tr></thead><tbody><tr><td>Codec: Realtek ALC3246</td><td>是告诉了你的声卡型号是 ALC3246(ALC256)</td></tr><tr><td>Address: 0</td><td>会告诉你生成 configdata 的数据的前缀是 0, 比如上面显示输出信息最后一行的<code>Modified Verbs in One Line:</code>后面所有的数据中, 每组数据的第一位就是这个 <code>0</code> , 如果<code>Address: 2</code>, 那么每组数据的第一位就是 <code>2</code> , 这个后面我们会用到它</td></tr><tr><td>DevID: 283902550 (0x10ec0256)</td><td>283902550 是 0x10ec0256 的 10 进制值, 0x10ec 是指 vendorID(芯片供应商 ID), 是 REALTEK 的设备标识, 0256 是指型号, 去掉前面的 0, 您的声卡型号就是: ALC256</td></tr></tbody></table>

过去的教程会告诉你, 通过运行`verbit.sh`整理出来的节点是:

```
18 0x12 Mic at Int
20 0x14 Speaker at Int
33 0x21 HP Out at Ext Front
```

至于上面输出信息里显示的 0x13 0x1d 都是无效的节点. 而且它只有三个有效的节点, 我的声卡 ALC256 是用于笔记本, 按理说它至少还应该存在另一个 Mic Ext 的节点, 我需要找出它来. 看来使用传统的方法已经无法满足我的求知欲了.  
既然使用 Linux 可以提取 codec, 那么有没有可能

呢?  
为了测试我的声卡存在第四个节点, 我需要找出耳机的 Mic 输入节点, 于是乎重新引导进入 Linux, 插上耳机, 它弹出了这个窗口:  
[![](http://7.daliansky.net/SelectAudioDevice.png)](http://7.daliansky.net/SelectAudioDevice.png)  
这三个选项的意思是让我选择声音设备, 既然我希望找出耳机麦克风的输入, 我就选择了中间带有耳麦的图标  
进入`Sound Settings`, 点击 `Input`, 选择`Headset Microphone`, 调节`Input volume`, 对着耳麦说话, 发现有输入电平了, 再切换到内置麦克风`Internal Microphone`, 也有输入电平, 那么至少说明我的声卡功能是完整的, 至少在 Linux 下它工作的很好.  
[![](http://7.daliansky.net/SoundInputDeviceSelect.png)](http://7.daliansky.net/SoundInputDeviceSelect.png)  
测试结果已经出来了, 我想要知道耳麦的有效节点是什么, 使用组合键`CTRL+ALT+t`打开终端, 输入命令:`dmesg`看看它会有什么变化, 结果我就看到了下面的文字:

```
[    5.040591] snd_hda_codec_realtek hdaudioC0D0: autoconfig for ALC3246: line_outs=1 (0x14/0x0/0x0/0x0/0x0) type:speaker
[    5.040593] snd_hda_codec_realtek hdaudioC0D0:    speaker_outs=0 (0x0/0x0/0x0/0x0/0x0)
[    5.040595] snd_hda_codec_realtek hdaudioC0D0:    hp_outs=1 (0x21/0x0/0x0/0x0/0x0)
[    5.040595] snd_hda_codec_realtek hdaudioC0D0:    mono: mono_out=0x0
[    5.040596] snd_hda_codec_realtek hdaudioC0D0:    inputs:
[    5.040598] snd_hda_codec_realtek hdaudioC0D0:      Headset Mic=0x19
[    5.040599] snd_hda_codec_realtek hdaudioC0D0:      Headphone Mic=0x1a
[    5.040600] snd_hda_codec_realtek hdaudioC0D0:      Internal Mic=0x12
```

我不想看到其它的输出信息, 我只需要得到我想要的信息, 于是将命令改为:

```
sky@sky-Inspiron-7560:~$ dmesg | grep snd_hda_codec_realtek
[    5.040591] snd_hda_codec_realtek hdaudioC0D0: autoconfig for ALC3246: line_outs=1 (0x14/0x0/0x0/0x0/0x0) type:speaker
[    5.040593] snd_hda_codec_realtek hdaudioC0D0:    speaker_outs=0 (0x0/0x0/0x0/0x0/0x0)
[    5.040595] snd_hda_codec_realtek hdaudioC0D0:    hp_outs=1 (0x21/0x0/0x0/0x0/0x0)
[    5.040595] snd_hda_codec_realtek hdaudioC0D0:    mono: mono_out=0x0
[    5.040596] snd_hda_codec_realtek hdaudioC0D0:    inputs:
[    5.040598] snd_hda_codec_realtek hdaudioC0D0:      Headset Mic=0x19
[    5.040599] snd_hda_codec_realtek hdaudioC0D0:      Headphone Mic=0x1a
[    5.040600] snd_hda_codec_realtek hdaudioC0D0:      Internal Mic=0x12
```

得到的输出结果跟上面的相同, 于是我发现了在 Linux 下我的声卡的有效节点分别为:

*   0x14 Speaker out
*   0x21 HP out
*   0x19 Headset Mic in
*   0x1a Headphone Mic in
*   0x12 Internal Mic in

暂且不讨论我的声卡是几节点, 我发现通过 linux 完全可以得出声卡的有效节点, 而且这种方式好像也不需要使用什么专用的工具, 只是在 linux 下面通过内部命令`dmesg`无意中就被我发现了声卡中最挠头的找有效节点的打怪秘籍, 这种方法在整个 Hackintosh 社区里还没见有人用过. 看来这跟我之前使用 linux 经常会通过`dmesg`排错的使用习惯是分不开的, 这里要由衷地给自己点一个大大的

在这个特别的日子里`11.11`, 没有陪女朋友的小伙伴们, 都可以使用我刚 get 到的新技能愉快地打怪升级啦!

暂更于此, 做个记号 2017 年 11 月 11 日

继续更新

先将节点整理成一个表格, 这个表格里会包括之前整理出的数据

<table><thead><tr><th>有效节点</th><th>10 进制</th><th>设备名称</th></tr></thead><tbody><tr><td>0x14</td><td>20</td><td>Speaker out</td></tr><tr><td>0x21</td><td>33</td><td>HP out</td></tr><tr><td>0x19</td><td>25</td><td>Headset Mic in</td></tr><tr><td>0x1a</td><td>26</td><td>Headphone Mic in</td></tr><tr><td>0x12</td><td>18</td><td>Internal Mic in</td></tr></tbody></table>

之所以加上转换后的 10 进制, 就是为了以后制作 pathmap 做准备

使用 codecgraph 生成 pathmap 图  
命令如下:

```
$ codecgraph codec\#0
Codec: Realtek ALC3246
Generating codec#0.svg
```

如果出现`error: dot executable not found (did you install graphviz?)`的错误提示的话, 那么还需要执行下面的动作, 打开终端, 输入命令:

```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)" # 安装brew
```

安装`brew`的过程中会提醒你安装`Xcode Command Tools`, 请选择`yes`.  
安装完`brew`之后, 继续执行:

```
brew install graphviz   # 安装codecgraph所需要的依赖程序
```

这个生成的 codec#0.svg 就是声卡的 pathmap, 使用`Sketch.app`打开它  
[![](http://7.daliansky.net/codec.png)](http://7.daliansky.net/codec.png)  
是不是看了头老大? 没错, 我刚开始看的时候也是一头的雾水, 看多了后就会总结出经验来  
还记得我们在前面整理出的有效节点吧?!  
还是以我的声卡为例, 上文中我已经整理出 5 个节点, 按照顺序来:  
0x14, 这个是 Speaker 的节点, 也就是喇叭, 我们也俗称为外放, 如果想让它正确地发声, 就需要看看跟它连接的节点都有哪些, 在图片里找到 0x14 位于图片的最下方, 我把它放大下:  
[![](http://7.daliansky.net/node_output.png)](http://7.daliansky.net/node_output.png)  
通过观察我们会发现, 0x14 与 0x02 连接, 转换成 10 进制就是 20->2, 同样的, 0x21 与 0x03 连接, 转换成 10 进制就是 33->3, 我们把它放到表格中

<table><thead><tr><th>有效节点</th><th>10 进制</th><th>设备名称</th><th>路径</th></tr></thead><tbody><tr><td>0x14</td><td>20</td><td>Speaker out</td><td><strong>20-&gt;2</strong></td></tr><tr><td>0x21</td><td>33</td><td>HP out</td><td><strong>33-&gt;3</strong></td></tr><tr><td>0x19</td><td>25</td><td>Headset Mic in</td><td></td></tr><tr><td>0x1a</td><td>26</td><td>Headphone Mic in</td><td></td></tr><tr><td>0x12</td><td>18</td><td>Internal Mic in</td><td></td></tr></tbody></table>

怎么样? 也没想像的那么复杂吧? 两个输出设备的节点和路径已经整理出来, 下一步, 我们要把输入设备和节点整理出来  
[![](http://7.daliansky.net/node_input.png)](http://7.daliansky.net/node_input.png)  
再来看图片的上半部, 最右侧的 0x08 和 0x09 是两个声音输入的节点, 0x23 和 0x22 是两个混音设备, 最前方是设备节点, 这个就是输入设备的路径.  
我的声卡的 0x12 声音输入路径包括了 3 个节点; 为了便于理解, 被我圈起来的红线部分就是 0x12(Mic in) 的路径 (pathmap),8->35->18 就是路径, 照样还是要将数据放到表格里  
[![](http://7.daliansky.net/node12_pathmap.png)](http://7.daliansky.net/node12_pathmap.png)

<table><thead><tr><th>有效节点</th><th>10 进制</th><th>设备名称</th><th>路径</th></tr></thead><tbody><tr><td>0x14</td><td>20</td><td>Speaker out</td><td><strong>20-&gt;2</strong></td></tr><tr><td>0x21</td><td>33</td><td>HP out</td><td><strong>33-&gt;3</strong></td></tr><tr><td>0x19</td><td>25</td><td>Headset Mic in</td><td></td></tr><tr><td>0x1a</td><td>26</td><td>Headphone Mic in</td><td></td></tr><tr><td>0x12</td><td>18</td><td>Internal Mic in</td><td><strong>8-&gt;35-&gt;18</strong></td></tr></tbody></table>

还记得文章前面通过 linux 找出来的有效节点不? 我的耳麦可是支持输入的, 节点也找出来了, 是 0x19(Headset Mic in), 我需要将路径找出来, 我发现 0x19 同时连接了两个节点: 0x23 和 0x22, 相应地路径为: 8->35->25 和 9->34->25, 顺手也把 0x1a 的路径也整理出来备用, 将这两组数据补充进表格里

<table><thead><tr><th>有效节点</th><th>10 进制</th><th>设备名称</th><th>路径</th></tr></thead><tbody><tr><td>0x14</td><td>20</td><td>Speaker out</td><td><strong>20-&gt;2</strong></td></tr><tr><td>0x21</td><td>33</td><td>HP out</td><td><strong>33-&gt;3</strong></td></tr><tr><td>0x19</td><td>25</td><td>Headset Mic in</td><td><strong>8-&gt;35-&gt;25</strong> <strong>9-&gt;34-&gt;25</strong></td></tr><tr><td>0x1a</td><td>26</td><td>Headphone Mic in</td><td><strong>8-&gt;35-&gt;26</strong> <strong>9-&gt;34-&gt;26</strong></td></tr><tr><td>0x12</td><td>18</td><td>Internal Mic in</td><td><strong>8-&gt;35-&gt;18</strong></td></tr></tbody></table>

至于 0x1a 是否有效暂且放到一边, 一个包括了有效节点 / 设备名称 / 路径的表格就整理完成了, 这里 *** 需要强调的一点是: 路径数值使用 10 进制 ***

> `ConfigData`的获得有两种方法: 这两种方法分别为: 通过`codec`里面的`Pin Default`提取以及通过`PinConfigs`提取, 然后修正`ConfigData`. 下面我们将分别介绍这两种提取方法

[](#通过pinconfigs提取configdata)通过 PinConfigs 提取 ConfigData
--------------------------------------------------------

ConfigData 数据位于`PinConfigs.kext/Contents/Info.plist`里,`PinConfigs.kext`作为`AppleALC`插件被加载.  
用到的工具:`Plist Editor Pro.app`  
还是以我的 ALC256 为例: 打开`Info.plist`后, 使用组合快捷键`⌘+f`打开搜索: `ALC256` 或者 `283902550` , 它会直接定位到你需要查找的数据位置, 再按`⌘+g`, 将相同的字符串全部搜索完毕, 直到它又跳转到第一个搜索结果的位置, 结果如下图:  
[![](http://7.daliansky.net/PlistEditorPro.png)](http://7.daliansky.net/PlistEditorPro.png)  
ALC256 的 ConfigData 的数据有三组, 分别对应的 LayoutID 为: 13 28 56  
将这三组 ConfigData 复制出来:

id:13  
`01271C10 01271D00 01271EA6 01271F90 01471C20 01471D00 01471E17 01471F90 01971C30 01971D10 01971E8B 01971F02 02171C50 02171D10 02171E2B 02171F02 01470c02`

id:28  
`01471C50 01471D00 01471E17 01471F90 01470c02 01A71C30 01A71D00 01A71EA0 01A71F90 02171C20 02171D10 02171E21 02171F00`

id:56  
`01271C10 01271D00 01271EA6 01271F90 01471C20 01471D00 01471E17 01471F90 01971C30 01971D10 01971E8B 01971F02 01A71C40 01A71D10 01A71E81 01A71F02 01D71CF0 01D71D11 01D71E11 01D71F41 02171C50 02171D10 02171E2B 02171F02 01470c02 02170c02`

这么看起来是杂乱无章的, 我们将它格式化一下, 每个节点一行, 四组数据:  
id:13

```
01271C10 01271D00 01271EA6 01271F90 
01471C20 01471D00 01471E17 01471F90 
01971C30 01971D10 01971E8B 01971F02 
02171C50 02171D10 02171E2B 02171F02 01470c02
```

id:28

```
01471C50 01471D00 01471E17 01471F90 01470c02 
01A71C30 01A71D00 01A71EA0 01A71F90 
02171C20 02171D10 02171E21 02171F00
```

id:56

```
01271C10 01271D00 01271EA6 01271F90 
01471C20 01471D00 01471E17 01471F90 01470c02 
01971C30 01971D10 01971E8B 01971F02 
01A71C40 01A71D10 01A71E81 01A71F02 
01D71CF0 01D71D11 01D71E11 01D71F41 
02171C50 02171D10 02171E2B 02171F02 02170c02
```

从中我们可以发现:  
id:13 为四行四节点, 节点为:`0x12 0x14 0x19 0x21`  
id:28 为三行三节点, 节点为:`0x14 0x1A 0x21`  
id:56 为六行六节点, 节点为:`0x12 0x14 0x19 0x1A 0x1D 0x21`  
现在有个问题摆在你们的面前, 我上面明明刚说过每个节点为四组数据, 那为什么在 0x14 和 0x21 的节点后面又多出来一组数据呢? 这组数据又有什么作用呢? 原来秘密都藏在`codec`里面, 它定义了某些输出设备具有`EAPD`, 比如 0x14 节点的数据:

```
Node 0x14 [Pin Complex] wcaps 0x40058d: Stereo Amp-Out
  Control: , index=0, device=0
    ControlAmp: chs=3, dir=Out, idx=0, ofs=0
  Amp-Out caps: ofs=0x00, nsteps=0x00, stepsize=0x00, mute=1
  Amp-Out vals:  [0x00 0x00]
  Pincap 0x00010014: OUT EAPD Detect
  EAPD 0x2: EAPD
  Pin Default 0x90170120: [Fixed] Speaker at Int N/A
    Conn = Analog, Color = Unknown
    DefAssociation = 0x2, Sequence = 0x0
    Misc = NO_PRESENCE
  Pin-ctls: 0x40: OUT
  Unsolicited: tag=00, enabled=0
  Power states:  D0 D1 D2 D3 EPSS
  Power: setting=D0, actual=D0
  Connection: 1
     0x02
```

具有`EAPD`的节点需要添加参数`SET_EAPD_BTLENABLE`, 转换成数值为`0x70c`, 完整的数值为:`Address`+`节点`+`70c`+`02`, 即:`01470c02`, 它通过`hda-verb`可以执行, 比如`CodecCommander`就是调用的`hda-verb`执行的命令. 更多的`hda-verb`参数请参考我之前的文章:[hda-verb 参数详表](https://blog.daliansky.net/hda-verb-parameter-detail-table.html)  
这里我教大家一个简单判断`EAPD`节点的方法: 那就是它通常会位于`Speaker Out`和`HP Out`这两个输出节点上. 至于其它教程提到过的**关于`01470c02`是组神奇的代码, 可以让外放发声**的说法是错误的, 它可能刚好声卡的`Speaker Out`的输出节点是`0x14`而已. 如果您的`Speaker Out`输出节点是`0x16`, 那么就需要把它修改为`01670c02`, 当然要遵守这个公式:`Address`+`节点`+`71c`+`02`

本节教程先更新到这里 11-13-2017 10:25PM

续更

另外一种找出`EAPD`节点的方法: 使用工具`node_dump.sh`, 它会显示出每个节点的信息, 包括`EAPD`输出, 详细的用法以后再补充上.

说了半天, 也只说了一个`01470c02`是干什么用的, 其它的四组数据都分别代表什么呢? 这个话题如果展开讲的话, 没个一两天是说不明白的.  
接下来的部分就会介绍到这四组数据是怎么整理出来的.

[](#通过codec里面的pin-default提取并整理)通过`codec`里面的`Pin Default`提取并整理
-------------------------------------------------------------

将使用`verbit.sh`整理出来的`ALC256_dump.txt`重新打开:

```
Verbs from Linux Codec Dump File: codec#0

Codec: Realtek ALC3246   Address: 0   DevID: 283902550 (0x10ec0256)

   Jack   Color  Description                  Node     PinDefault             Original Verbs
--------------------------------------------------------------------------------------------------------
Digital Unknown  Mic at Int N/A              18 0x12   0x90a60170   01271c70 01271d01 01271ea6 01271f90
Unknown Unknown  Line Out at Ext N/A         19 0x13   0x40000000   01371c00 01371d00 01371e00 01371f40
 Analog Unknown  Speaker at Int N/A          20 0x14   0x90170140   01471c40 01471d01 01471e17 01471f90
    1/8   Black  Speaker at Ext Rear         24 0x18   0x411111f0   01871cf0 01871d11 01871e11 01871f41
    1/8   Black  Speaker at Ext Rear         25 0x19   0x411111f0   01971c10 01971d20 01971e81 01971f02
    1/8   Black  Speaker at Ext Rear         26 0x1a   0x411111f0   01a71cf0 01a71d11 01a71e11 01a71f41
    1/8   Black  Speaker at Ext Rear         27 0x1b   0x411111f0   01b71cf0 01b71d11 01b71e11 01b71f41 
 Analog    Pink  Modem Hand at Ext N/A       29 0x1d   0x40779a2d   01d71c2d 01d71d9a 01d71e77 01d71f40
    1/8   Black  Speaker at Ext Rear         30 0x1e   0x411111f0   01e71cf0 01e71d11 01e71e11 01e71f41
    1/8   Black  HP Out at Ext Front         33 0x21   0x02211030   02171c30 02171d10 02171e21 02171f02
--------------------------------------------------------------------------------------------------------
```

将数据整理成如下的样式:

*   [Fixed] 是内部设备
*   [Jack] 是通过插孔进行连接的外部设备
*   [N/A] 是其它未知设备

```
Node	PinDefault		Description 
----------------------------------------------------------
0x12	0x90a60170		[Fixed]	Mic at Int       
0x13	0x40000000		[N/A]	Line Out at Ext N/A  
0x14	0x90170140		[Fixed]	Speaker at Int   
0x18	0x411111f0		[N/A]	Speaker at Ext Rear  
0x19	0x411111f0		[N/A]	Speaker at Ext Rear  
0x1a	0x411111f0		[N/A]	Speaker at Ext Rear  
0x1b	0x411111f0		[N/A]	Speaker at Ext Rear  
0x1d	0x40779a2d		[N/A]	Modem Hand at Ext N/A
0x1e	0x411111f0		[N/A]	Speaker at Ext Rear  
0x21	0x02211030		[Jack]	HP Out at Ext Front  
----------------------------------------------------------
```

将`PinDefault`进行小端转换 [little-endian](https://zh.wikipedia.org/zh-hans/%E5%AD%97%E8%8A%82%E5%BA%8F), 这里借用一张紫米教程里面的图片进行小端转换的理解:  
[![](http://7.daliansky.net/little-endian.jpg)](http://7.daliansky.net/little-endian.jpg)  
就是把`PinDefault`的数据两两对调, 比如:`0x12`节点的`[Fixed] Mic at Int`原始数据为:`0x90a60170`, 去掉前面的`0x`, 变成`90 a6 01 70`,  
最前面的`90`跑到了最后, 最后的`70`又跑到了最前面, 中间的`a6`和`01`再对调下, 就变成了`70 01 a6 90`. 整理后, 变成如下的格式:

```
Node            c  d  e  f              Description
12		70 01 a6 90		[Fixed]	Mic at Int       
13		00 00 00 40		[N/A]	Line Out at Ext N/A  
14		40 01 17 90		[Fixed]	Speaker at Int   
18		f0 11 11 41		[N/A]	Speaker at Ext Rear  
19		f0 11 11 41		[N/A]	Speaker at Ext Rear  
1a		f0 11 11 41		[N/A]	Speaker at Ext Rear  
1b		f0 11 11 41		[N/A]	Speaker at Ext Rear  
1d		2d 9a 77 40		[N/A]	Modem Hand at Ext N/A
1e		f0 11 11 41		[N/A]	Speaker at Ext Rear  
21		30 10 21 02		[Jack]	HP Out at Ext Front
```

下一步, 我们要对数据进行修正, 让它看起来更像`APPLE`.

[](#修正pindefault并生成configdata)修正`PinDefault`, 并生成`ConfigData`
-------------------------------------------------------------

上一节我们只是把数据进一步地整理出来, 并进行了格式化, 我们并没有对数据进行修正. 本节我们就开始进行`PinDefault`的修正, 先上一张我整理出来的表格:  
[![](http://7.daliansky.net/pinconfigs.png)](http://7.daliansky.net/pinconfigs.png)

*   前面我提到过通过 Linux 找到的节点`0x19`, 它在 Linux 下面显示的设备为`Headset Mic`, 在上一节中我们整理出的`0x19`的设备为:`[N/A] Speaker at Ext Rear` , `PinDefault`值为:`f0 11 11 41`, 通过上表我们得知, 这个端口是被屏蔽掉的, 就是不起作用的节点.
*   现在我们希望它能正常工作, 我的想法是把它修正为:`[Jack] Mic at Ext`, 一个可以正常工作的外部麦克输入设备, 结合上面的表格, 我将`f0 11 11 41`修改为:`70 20 a1 02`.
*   由于我的耳机插孔只有一个, 也就是俗称的二合一插孔, 我希望在上面的外麦克工作不正常的情况下, 可以将耳麦和耳机输出定义成组合插孔 (2in1), 上表中`Connection Type`的连接类型中, 正好有个`B=Combination`, 它的意思就是:`组合`, 我需要同时修正耳麦和耳机的数值:
    *   `Mic at Ext` : `f0 11 11 41`->`70 20 ab 02`
    *   `HP Out Ext` : `30 10 21 02`->`30 10 2b 02`

[](#屏蔽无效节点f0-00-00-40)屏蔽无效节点:`f0 00 00 40`
------------------------------------------

屏蔽掉无效的节点, 避免产生杂音和底噪, 更详细的描述请见`其它问题`

`0x1d`的节点为无效节点, 所以必须把它屏蔽起来, 为了让它更符合`APPLE`的规范, 我们使用`f0 00 00 40`来屏蔽, 包括其它的不用的节点. 修正后的数据为:

```
Node            c  d  e  f              Description
12		70 01 a6 90		[Fixed]	Mic at Int       
13		f0 00 00 40		[N/A]	Line Out at Ext N/A  
14		40 01 17 90		[Fixed]	Speaker at Int   
18		f0 00 00 40		[N/A]	Speaker at Ext Rear  
19		70 10 ab 02		[Jack]	Mic at Ext  
1a		f0 00 00 40		[N/A]	Speaker at Ext Rear  
1b		f0 00 00 40		[N/A]	Speaker at Ext Rear  
1d		f0 00 00 40		[N/A]	Modem Hand at Ext N/A
1e		f0 00 00 40		[N/A]	Speaker at Ext Rear  
21		30 10 2b 02		[Jack]	HP Out at Ext Front
```

现在, 我们可以生成最终的`ConfigData`了,  
`ConfigData`的计算公式为:

```
Final Config:
Address + Node + 71c +【c】
Address + Node + 71d +【d】
Address + Node + 71e +【e】
Address + Node + 71f +【f】
```

这里的`c` `d` `e` `f`就是上面整理出来的 2 位数值  
我的 ALC256 的`Address`值为 0, 现在我将节点`0x12`的`ConfigData`值通过上面的公式整理后得出:

```
01271c70 01271d01 01271ea6 01271f90
```

相应地, 其它的节点整理出来的数据如下:

```
01271c70 01271d01 01271ea6 01271f90
01371cf0 01371d00 01371e00 01371f40
01471c40 01471d01 01471e17 01471f90 01470c02
01871cf0 01871d00 01871e00 01871f40
01971c70 01971d10 01971eab 01971f02
01a71cf0 01a71d00 01a71e00 01a71f40
01b71cf0 01b71d00 01b71e00 01b71f40
01d71cf0 01d71d00 01d71e00 01d71f40
01e71cf0 01e71d00 01e71e00 01e71f40
02171c30 02171d10 02171e2b 02171f02 02170c02
```

我将编辑的过程做了段视频, 方便大家理解.

打开`Sublime Text.app`, 按住鼠标中键打开块复制, 将`c` `d` `e` `f`下面的数据块拷贝一份到最下面空白处, 在第一列按鼠标中键一下拉到下面, 输入`Address`值`0`, 再把上方的`Node`下面的数据做块复制, 粘贴到 0 后面, 再输入:`71c`, 再将`01271`做块复制到后面, 再分别输入`d` `e` `f`即可. 最后在`0x14`和`0x21`节点后面添加两组`EAPD`数据.  
将上面整理出来的数据去掉每行的硬回车后得到的一组数据就是最终的`ConfigData`  
`01271c70 01271d01 01271ea6 01271f90 01371cf0 01371d00 01371e00 01371f40 01471c40 01471d01 01471e17 01471f90 01470c02 01871cf0 01871d00 01871e00 01871f40 01971c70 01971d10 01971eab 01971f02 01a71cf0 01a71d00 01a71e00 01a71f40 01b71cf0 01b71d00 01b71e00 01b71f40 01d71cf0 01d71d00 01d71e00 01d71f40 01e71cf0 01e71d00 01e71e00 01e71f40 02171c30 02171d10 02171e2b 02171f02 02170c02`

到这里,`ConfigData`的数据就算整理完了.

这一节里, 我们就来一起动手找出最适合你的注入 id  
正确的注入 id 可以保证您的声卡各项功能都能正常使用, 它至关重要, 这里给出的方法也是为了进一步验证前面我们所做的工作.  
到找注入 id 这一步, 您至少已经有了这些数据:

*   声卡基础资料:

<table><thead><tr><th>名称</th><th>解释</th></tr></thead><tbody><tr><td>Codec: Realtek ALC3246</td><td>是告诉了你的声卡型号是 ALC3246(ALC256)</td></tr><tr><td>Address: 0</td><td>会告诉你生成 configdata 的数据的前缀是 0, 比如上面显示输出信息最后一行的<code>Modified Verbs in One Line:</code>后面所有的数据中, 每组数据的第一位就是这个 <code>0</code> , 如果<code>Address: 2</code>, 那么每组数据的第一位就是 <code>2</code> , 这个后面我们会用到它</td></tr><tr><td>DevID: 283902550 (0x10ec0256)</td><td>283902550 是 0x10ec0256 的 10 进制值, 0x10ec 是指 vendorID(芯片供应商 ID), 是 REALTEK 的设备标识, 0256 是指型号, 去掉前面的 0, 您的声卡型号就是: ALC256</td></tr></tbody></table>

*   整理出来的有效节点及路径

<table><thead><tr><th>有效节点</th><th>10 进制</th><th>设备名称</th><th>路径</th></tr></thead><tbody><tr><td>0x14</td><td>20</td><td>Speaker out</td><td><strong>20-&gt;2</strong></td></tr><tr><td>0x21</td><td>33</td><td>HP out</td><td><strong>33-&gt;3</strong></td></tr><tr><td>0x19</td><td>25</td><td>Headset Mic in</td><td><strong>8-&gt;35-&gt;25</strong> <strong>9-&gt;34-&gt;25</strong></td></tr><tr><td>0x1a</td><td>26</td><td>Headphone Mic in</td><td><strong>8-&gt;35-&gt;26</strong> <strong>9-&gt;34-&gt;26</strong></td></tr><tr><td>0x12</td><td>18</td><td>Internal Mic in</td><td><strong>8-&gt;35-&gt;18</strong></td></tr></tbody></table>

*   最终的`ConfigData`数据:  
    `01271c20 01271d01 01271ea6 01271f90 01371cf0 01371d00 01371e00 01371f40 01471c40 01471d01 01471e17 01471f90 01470c02 01871cf0 01871d00 01871e00 01871f40 01971c10 01971d10 01971e8b 01971f02 01a71cf0 01a71d00 01a71e00 01a71f40 01b71cf0 01b71d00 01b71e00 01b71f40 01d71cf0 01d71d00 01d71e00 01d71f40 01e71cf0 01e71d00 01e71e00 01e71f40 02171c30 02171d10 02171e2b 02171f02 02170c02`

下一步, 我们就要将这些数据`放`进`AppleALC`相应的位置, 让它发挥作用.

[](#下载applealc)下载`AppleALC`
---------------------------

操作步骤:

1.  创建`AppleALC`的本地仓库的克隆版本:
    
    *   使用`git`命令, 更多`git`命令的用法请参考 [git 使用简易指南](http://www.bootcss.com/p/git-guide/):
        
        ```
        mkdir ~/git # 在家目录下新建立`git`目录
        cd ~/git    # 进入`git`目录
        git clone https://github.com/vit9696/AppleALC   # 克隆AppleALC到本地
        ```
        
    *   或者通过浏览器下载 [AppleALC 最新版](https://github.com/vit9696/AppleALC/archive/master.zip)
        
2.  打开`AppleALC`目录, 进入`Resources`目录, 你会发现这里面`躺`着截止到目前所支持的声卡的全部型号. 我的声卡是`ALC256`, 我就点击`ALC256`为便于理解, 我将除了`ALC256`之外的其它型号的声卡目录全部删除了, 它看起来如下图所示:  
    [![](http://7.daliansky.net/ALC_Folder.png)](http://7.daliansky.net/ALC_Folder.png)我们会发现该目录下包括了一个定义文件`Info.plist`, 打开`Info.plist`, 你会发现, 它定义了一个声卡驱动所需要的数据,`CodecID`后面的`598`是`ALC256`的`0x256`的 10 进制数值,`CodecName`是声卡名称的描述, 这里是`ALC256(3246)`, 接下来是`Files`, 它分成两部分, 一部分是`Layouts`, 它定义声卡设备的布局, 另一部分是`Platforms`, 它定义声卡的平台注入, 包括有效节点和路径的定义.  
    [![](http://7.daliansky.net/ALC256_Info.plist.png)](http://7.daliansky.net/ALC256_Info.plist.png)
    

三个`layout`开头的文件以及三个`Platforms`开头的文件,`Platforms`为路径定义文件. 其中`layout13.xml.zlib` `layout28.xml.zlib` `layout56.xml.zlib`就是注入的 id, 只是你的声卡需要注入的是哪个 ID 才能获得最佳效果, 还需要分析其它的数据.

[](#如何找出适合你的id)如何找出适合你的 id
--------------------------

通过制作几十个声卡仿冒的驱动, 我发现找出注入的 id 还是有些窍门的, 比如说我可以通过`PinConfigs.kext`的`Info.plist`, 将同一型号的声卡的`ConfigData`都找出来, 方法:

*   根据上面整理的声卡基础资料将`Address`不为`0`的排除出去
*   剩下的`LayoutID`里, 再通过`ConfigData`将有效节点的数据留下, 无效的排除, 这样剩下来的`LayoutID`也就没几个了
*   最后, 通过`PlatformsID`, 找出有效节点及路径, 最终确定注入的 id
*   需要注意的是, 某些型号的声卡的注入 id 可以是唯一的, 也可以是多个, 这些都需要通过分析数据进行确认

> 这里插句题外话: 之前给群友做声卡仿冒的时候,`ALC269`和`ALC887`我都是直接绕路走的, 或者让群友用`VoodooHDA`, 或者直接拒绝. 现在可以使用排除法, 很快就能找到注入的 id

更新于: 11-21-2017 22:30

> 上文中我们提到了通过`PlatformsID`, 找出有效节点及路径, 最终确定注入的 id

如何确认路径是否正确, 我应该怎么做呢? 先打开`Platforms`文件, 需要用到的工具:`zlib转换器.app`.  
操作方法:

*   打开`zlib转换器.app`, 把需要解压缩的后缀为. zlib 的文件拖进窗口里, 按下`CONVERT`按钮, 就会在当前的目录下生成去掉了. zlib 后缀的可以编辑的. xml 文件了. 相应地, 当你编辑好了的. xml 文件, 拖进窗口里, 按下`CONVERT`按钮, 会在当前目录下生成. zlib 后缀的文件了.
    
    <video src="" control></video>
    

使用工具`PlistEdit Pro.app`打开`Platforms56.xml`, 依次打开`PathMap->0->0->0`, 这里的`NodeID`就是节点 id. 为便于理解, 我将`Mic Int`的路径做了说明, 如下图显示:  
[![](http://7.daliansky.net/Platforms_edit2.png)](http://7.daliansky.net/Platforms_edit2.png)

为什么要建立一个全新的 layout-id 呢? 很多时候, 我们其实仿冒声卡驱动, 只是为了让自己的声卡驱动正常, 而通常的做法是找出一个存在的 ID, 然后整理数据, 修改路径和`ConfigData`, 最后编译使用.

那如果`AppleALC`的维护者`vit9696`更新了源代码怎么办呢? 你总不能再重复做一遍驱动吧? 或者说你好不容易找人帮你做好了一个驱动程序, 然后`AppleALc`就升级了, 旧的驱动无法用到新的系统当中, 这个时候你的选择是什么? 继续找人帮你做驱动? 或者自己动手仿冒?

其实最好的方法是: 如果你的声卡不存在于 [AppleALC 支持的 Codecs 列表](https://blog.daliansky.net/AppleALC-Supported-codecs.html)中, 或者其中的某个注入 id 并不会很好地工作, 那么你就需要新建一个注入 id 编译使用, 测试无误后, 你甚至还可以将这些修改提交给`vit9696`, 他会将这些更新合并到`AppleALC`的源代码中, 这样以后再有更新你只需要下载个`AppleALC`的发行版直接使用就好了.  
[![](http://7.daliansky.net/ALC298_ID99_ALC295_ID13.png)](http://7.daliansky.net/ALC298_ID99_ALC295_ID13.png)

上图中, 是我给小米 Pro 制作的全新的 ALC298 注入 ID:99, 只要是小米 Pro 的机型, 可以直接注入 id:99 来驱动你的声卡.

如果你想制作属于你的专属声卡注入 ID, 可添加文章最后的 QQ 群, 进入群内一起讨论.

使用 xcode  
通过 lilu 联合编译

[](#检查applealc是否正确加载)检查`AppleALC`是否正确加载
---------------------------------------

打开终端, 输入命令:

```
kextstat | grep AppleALC
```

显示内容如下:

```
kextstat | grep AppleALC
   36    0 0xffffff7f8331b000 0xb0000    0xb0000
       as.vit9696.AppleALC (1.2.1) 
       7CDA2635-8BE4-36E1-B882-8CA3B90284F3 <35 7 5 4 3 2 1>
```

我们可以看到`AppleALC`驱动已经被正确加载, 版本是`1.2.1`, 如果没有内容输出, 那么就有可能没有加载. 通常这种情况是因为`Lilu`的版本引起的, 可以尝试使用`Lilu` Debug 版本.

[](#检查注入id是否正确)检查注入 id 是否正确
---------------------------

打开终端, 输入命令:

```
ioreg -l | grep layout-id
```

显示内容如下:

```
~ % ioreg -l | grep layout-id
    | |   | |   "layout-id" = <38000000>
    | |   |             | |   "IOAudioEngineDeviceDescription" = {
    "num-internal-mics"=1,
    "driver digital input gain"=0,
    "layout-id"=56,
    "post-processing input gain"=25,
    "VP-pregain"=25}
```

为了便于讲解, 我将内容做了换行处理.  
其中的`"layout-id"=56`就是指注入 id 是 `56` , 而`<38000000>`是`56`的 16 进制

[](#检查pinconfigurations)检查`PinConfigurations`
---------------------------------------------

使用`IORegistryExplorer.app`, 找到`HDEF@1F,3`, 右侧的`PinConfigurations`就是最终的四个节点的数值, 同时我们发现其它的节点已经不存在了, 这样的结果正是我们需要的.  
[![](http://7.daliansky.net/ioreg_pinconfigurations.png)](http://7.daliansky.net/ioreg_pinconfigurations.png)

[](#解决耳机切换睡眠唤醒去除底噪等常见问题)解决耳机切换 / 睡眠唤醒 / 去除底噪等常见问题
-------------------------------------------------

> 经常会有群友在使用`AppleALC`声卡仿冒驱动程序驱动了声卡后, 出现最多的问题就是外放正常, 耳机不正常, 具体表现为: 耳机在插入一半的时候有声音, 完全插入后却没有声音的现象; 另一种最常见的现象是开机声音正常, 睡眠唤醒后无声 / 出现杂音 / 有电流声 (底噪)/ 单声道 / 只有伴奏音, 没有人声等等的问题.
> 
> 本节教程就让我们来一起解决这些常见问题, 还你一个干净的, 没有杂音的声卡驱动.

在我们着手解决这些问题之前, 需要先说说这些问题是如何产生的:

让我们打开电脑, 进入`MacOS`系统后,`AppleALC`仿冒驱动会成功加载`AppleHDA`以驱动你的声卡, 这个时候声卡通常工作都是正常的. 为了证明`AppleALC`加载后的有效节点的状态, 我们需要使用两个[工具](https://github.com/daliansky/dell7000/tree/master/hda-tools)`widget_dump.sh`和`node_dump.sh`, 使用方法: 打开终端, 输入命令:

```
widget_dump.sh
```

显示的输出信息如下:

```
sky@skys-MacBookPro  ~  widget_dump.sh
Pin Widget Control
	nid = 0x02 --> result 0x00000000
	nid = 0x03 --> result 0x00000000
	nid = 0x04 --> result 0x00000000
	nid = 0x05 --> result 0x00000000
	nid = 0x06 --> result 0x00000000
	nid = 0x07 --> result 0x00000000
	nid = 0x08 --> result 0x00000000
	nid = 0x09 --> result 0x00000000
	nid = 0x0a --> result 0x00000000
	nid = 0x0b --> result 0x00000000
	nid = 0x0c --> result 0x00000000
	nid = 0x0d --> result 0x00000000
	nid = 0x0e --> result 0x00000000
	nid = 0x0f --> result 0x00000000
	nid = 0x10 --> result 0x00000000
	nid = 0x11 --> result 0x00000000
	nid = 0x12 --> result 0x00000020	# 内置麦克风
	nid = 0x13 --> result 0x00000000
	nid = 0x14 --> result 0x00000040	# 喇叭
	nid = 0x15 --> result 0x00000000
	nid = 0x16 --> result 0x00000000
	nid = 0x17 --> result 0x00000000
	nid = 0x18 --> result 0x00000000
	nid = 0x19 --> result 0x00000000	# 耳麦
	nid = 0x1a --> result 0x00000000
	nid = 0x1b --> result 0x00000000
	nid = 0x1c --> result 0x00000000
	nid = 0x1d --> result 0x00000000
	nid = 0x1e --> result 0x00000000
	nid = 0x1f --> result 0x00000000
	nid = 0x20 --> result 0x00000000
	nid = 0x21 --> result 0x00000000	# 耳机
	nid = 0x22 --> result 0x00000000
	nid = 0x23 --> result 0x00000000
	nid = 0x24 --> result 0x00000000
```

我们可以看到, 刚开机没有插入耳机的情况下, 只有外放的喇叭和内置的麦克风是工作的, 其它所有的节点的数据都是空白的.

当我们插入耳机后, 再输入命令:

```
widget_dump.sh
```

显示的输出信息如下:

```
sky@skys-MacBookPro  ~  widget_dump.sh
Pin Widget Control
	nid = 0x02 --> result 0x00000000
	nid = 0x03 --> result 0x00000000
	nid = 0x04 --> result 0x00000000
	nid = 0x05 --> result 0x00000000
	nid = 0x06 --> result 0x00000000
	nid = 0x07 --> result 0x00000000
	nid = 0x08 --> result 0x00000000
	nid = 0x09 --> result 0x00000000
	nid = 0x0a --> result 0x00000000
	nid = 0x0b --> result 0x00000000
	nid = 0x0c --> result 0x00000000
	nid = 0x0d --> result 0x00000000
	nid = 0x0e --> result 0x00000000
	nid = 0x0f --> result 0x00000000
	nid = 0x10 --> result 0x00000000
	nid = 0x11 --> result 0x00000000
	nid = 0x12 --> result 0x00000000	# 内置麦克风
	nid = 0x13 --> result 0x00000000
	nid = 0x14 --> result 0x00000000	# 喇叭	
	nid = 0x15 --> result 0x00000000
	nid = 0x16 --> result 0x00000000
	nid = 0x17 --> result 0x00000000
	nid = 0x18 --> result 0x00000000
	nid = 0x19 --> result 0x00000020	# 耳麦
	nid = 0x1a --> result 0x00000000
	nid = 0x1b --> result 0x00000000
	nid = 0x1c --> result 0x00000000
	nid = 0x1d --> result 0x00000000
	nid = 0x1e --> result 0x00000000
	nid = 0x1f --> result 0x00000000
	nid = 0x20 --> result 0x00000000
	nid = 0x21 --> result 0x000000c0	# 耳机
	nid = 0x22 --> result 0x00000000
	nid = 0x23 --> result 0x00000000
	nid = 0x24 --> result 0x00000000
```

这时我们会发现, 之前工作的喇叭和内置麦克风的数据已经发生了变化

<table><thead><tr><th>节点</th><th>名称</th><th>外放状态</th><th>耳机状态</th></tr></thead><tbody><tr><td>0x12</td><td>内置麦克风 (Mic Int)</td><td>0x000000<strong>20</strong></td><td>0x00000000</td></tr><tr><td>0x14</td><td>喇叭 (Speaker Out)</td><td>0x00000040</td><td>0x00000000</td></tr><tr><td>0x19</td><td>耳麦 (Mic Ext/Line In)</td><td>0x00000000</td><td>0x000000<strong>20</strong></td></tr><tr><td>0x21</td><td>耳机 (HP Out)</td><td>0x00000000</td><td>0x000000c0</td></tr></tbody></table>

通过上面的表格, 我们可以发现内部设备和外部设备正常工作的情况下会自动切换.

**widget_dump.sh 是调用`hda-verb`这个外部命令执行的,`hda-verb`依赖于`CodecCommander`, 所以请确认你已经加载了`CodecCommander`**

### [](#codeccommander-编解码器指挥官)CodecCommander _编解码器指挥官_

`CodecCommander`用于更新在扬声器或耳机节点上使用给定放大器的 HDA（高清晰度音频）编解码器上的 EAPD（外部放大器）状态（两者，甚至在某些情况下甚至是额外的）。 在 OSX 中，EAPD 在休眠状态下关闭，因此唤醒机器后音频仍然不起作用。

通常情况下，这个外部放大器出现在笔记本电脑和 ITX 板上，在使用 ALC269，ALC665 和类似编解码器的机器上最常见。 当机器进入睡眠状态时，功放处于关机状态，在唤醒之后，即使音频正常工作，扬声器 / 耳机仍然没有声音，因为功放需要发送一个编解码器命令才能启动。

最典型的两条命令是:

```
0x19 SET_PIN_WIDGET_CONTROL 0x24
0x21 SET_UNSOLICITED_ENABLE 0x82
```

其中:`SET_PIN_WIDGET_CONTROL`是作用于`Mic Ext`节点,`SET_UNSOLICITED_ENABLE`是作用于`HP Out`节点

[![](http://7.daliansky.net/Pin-ctls.png)](http://7.daliansky.net/Pin-ctls.png)

#### [](#pin-widget-control)Pin Widget Control

关于`Pin Widget Control`的描述请参阅[英特尔 HDA 规范 7.3.3.13](https://www.intel.com/content/dam/www/public/us/en/documents/product-specifications/high-definition-audio-specification.pdf)

Enable VRef

<table><thead><tr><th></th><th>Verb ID</th><th>Payload (8 Bits)</th><th>Response (32 Bits)</th></tr></thead><tbody><tr><td>Get</td><td>F07h</td><td>0</td><td>Bits 31:8 are 0<br>Bits 7:0 are PinCntl</td></tr><tr><td>Set</td><td>707h</td><td>Bits 7:0 are PinCntl</td><td>0</td></tr></tbody></table>

PinCntl 定义:

<table><thead><tr><th>7</th><th>6</th><th>5</th><th>4:3</th><th>2</th><th>1:0</th></tr></thead><tbody><tr><td>H-Phn Enable</td><td>Out Enable</td><td>In Enable</td><td>Rsvd</td><td>VrefEn[2]</td><td>VRefEn[1:0]/EPT</td></tr></tbody></table>

VRefEn 值:

<table><thead><tr><th>VRefEn Encoding</th><th>VREF Signal Level</th></tr></thead><tbody><tr><td>000b</td><td>Hi-Z</td></tr><tr><td>001b</td><td>50%</td></tr><tr><td>010b</td><td>Ground (0 V)</td></tr><tr><td>011b</td><td>Reserved</td></tr><tr><td>100b</td><td>80%</td></tr><tr><td>101b</td><td>100%</td></tr><tr><td>110b-111b</td><td>Reserved</td></tr></tbody></table>

SET_PIN_WIDGET_CONTROL:

<table><thead><tr><th></th><th></th><th></th><th></th><th>7654 3210</th><th></th><th></th></tr></thead><tbody><tr><td>ALC256</td><td></td><td></td><td></td><td></td><td></td><td></td></tr><tr><td>0x19</td><td>707</td><td>24</td><td>-&gt;</td><td>0010 0100</td><td>-&gt;</td><td>In Enable,VrefEn 80%</td></tr><tr><td>0x21</td><td>707</td><td>C0</td><td>-&gt;</td><td>1100 0000</td><td>-&gt;</td><td>Headphone Enable,Out Enable</td></tr></tbody></table>

SET_PIN_WIDGET_CONTROL 命令最终格式为:

`Address`+`NodeID`+707+`Pin-ctls`

最终的命令为:`01970724`

关于这部分的讨论请参阅 [Any fix to the Garbled/Distorted Headphone issue?](https://www.tonymacx86.com/threads/any-fix-to-the-garbled-distorted-headphone-issue.159031/page-3)

> 原引:
> 
> RehabMan 对于 VrefEn 和 EPT 的部分讨论内容:
> 
> 我也尝试了节点 0x19 的 0x20。 这也工作。 所以不知道在我的情况下是否需要 VrefEn 或 EPT。 但是他们与巫毒有关，所以我使用了这个价值。我没有看过 Voodoo 是否明确地设置了这些值，或者是复位后的状态结果。
> 
> 我认为我的设置 0xC0 到 0x21 也是没有必要的（仍在试验）。 节点 0x19 不在任何路径图上，所以我对于如何知道节点需要调整有些困惑。 尽管它在 Linux 转储中有连接到路径图上的其他节点。 进一步造成神秘，它被描述为 Linux 转储中的外部麦克风插孔。 为什么麦克风上的控件配置会影响耳机，这是一个谜。
> 
> 还要注意，VrefEn 和 EPT 之间有一些奇怪的重叠，这在规范中是不清楚的（它们似乎都使用相同的位）。 例如，将 VRefEn 设置为 001b 或 010b（50％/ 地面）将设置标记为 “保留” 的 EPT 编码 01b / 10b。 它确实说 EPT 是特定于“关联的数字 Pin Widget”，所以也许这个重叠解释了 VrefEn 不适用于这样的小部件。 它还指出 VRefEn 的有效值取决于 PinCaps，所以推测数字 Pin 小部件的 PinCap 不包含任何有效的 VRefEn 值。
> 
> 如果你沿着这条路走下去，你应该计划阅读大量的东西，让整个过程神秘化。

#### [](#unsolicited-response-未经请求的响应)Unsolicited Response 未经请求的响应

[![](http://7.daliansky.net/Unsolicited.png)](http://7.daliansky.net/Unsolicited.png)

关于`Unsolicited Response`的描述请参阅[英特尔 HDA 规范 7.3.3.14](https://www.intel.com/content/dam/www/public/us/en/documents/product-specifications/high-definition-audio-specification.pdf)

非请求响应控件确定节点是否被允许发送主动提供的响应，以及该标签将用于响应。此控制仅适用于支持主动响应的节点，如 Function Group Type 参数（第 7.3.4.4 节）和 Audio Widget Capabilities 参数（第 7.3.4.6 节）中所声明的。在获取或设置此控件之前，应该查询节点以确定它是否支持未经请求的响应。

Connection Select Control 连接选择控制:

<table><thead><tr><th></th><th>Verb ID</th><th>Payload (8 Bits)</th><th>Response (32 Bits)</th></tr></thead><tbody><tr><td>Get</td><td>F08h</td><td>0</td><td>Bits 31:8 are 0<br>EnableUnsol is bits 7:0</td></tr><tr><td>Set</td><td>708h</td><td>EnableUnsol is bits 7:0</td><td>0</td></tr></tbody></table>

EnableUnsol 格式:

<table><thead><tr><th>7</th><th>6</th><th>5:0</th></tr></thead><tbody><tr><td>Enable</td><td>0</td><td>Tag</td></tr></tbody></table>

设定值是 8bits 的一个数。记作 a7 a6 a5 a4 a3 a2 a1 a0.  
a7=1，表示 enabled。  
a6=0，没具体应用，不用管。  
a5-a0，存放 tag，需要将 tag 的值用 6 为数的二进制值表示  
tag=1（十进制）=000001（二进制）  
则 a7 a6 a5 a4 a3 a2 a1 a0=10000001  
a7 a6 a5 a4=1000（二进制）=8（十进制）  
a3 a2 a1 a0=0001（二进制）=1（十进制）  
则设定值 = 81  
最后加得到的设定值加 1，即 82

`SET_UNSOLICITED_ENABLE`命令最终格式为:

`Address`+`NodeID`+708+`Unsolicited`

最终的命令为:

`02170882`

目前`CodecCommander`所起的作用有限, 我基本上都忽略了`CodecCommander`的存在, 因为它并不能从根本上解决这些问题. 在我完善我的`ALC256`声卡驱动期间, 我还试用过`EAPDFix`, 最后都不得不放弃治疗. 直到我遇到了`ALCPlugFix`, 它能解决几乎所有的耳机切换问题.

### [](#alcplugfix能做什么)ALCPlugFix 能做什么?

*   它可以解决耳机插拔状态的切换
*   它是通过使用命令:`hda-verb 0xNode SET_PIN_WIDGET_CONTROL 0xVerbs`的方式进行状态切换
    *   `hda-verb`的由来
    *   `hda-verb`是 linux 下面的`alsa-project`的一条命令，它的作用是发送 HD-audio 命令

### [](#alcplugfix如何使用)ALCPlugFix 如何使用?

*   将`ALCPlugFix`同步到本地
    
    ```
    git clone https://github.com/daliansky/ALCPlugFix
    ```
    
*   打开`ALCPlugFix`目录下面的`main.m`, 将包含`hda-verb`语句的几行修改成如图所示  
    [![](http://7.daliansky.net/ALCPlugFix_ALC256.png)](http://7.daliansky.net/ALCPlugFix_ALC256.png)
    
*   其中`0x12`是`Mic Int`节点,`0x19`是`Mic Ext`节点, 之所以要这么修改, 是因为我的`ALC256`的`0x19`节点总是无法工作, 在插入耳机的时候我希望内置的`Mic Int`可以工作的同时还可以使用耳机.
    
*   将`XCODE`编译生成的`ALCPlugFix`文件复制到`alc_fix`目录, 然后双击`install双击自动安装.command`, 输入用户密码即可.
    

[ALCPlugFix](https://github.com/daliansky/ALCPlugFix) 工具下载

> 刚才我前面讲了,`ALCPlugFix`是依赖于`CodecCommander`执行的, 那我为什么不介绍下`CodecCommander`的工作原理呢, 其实说白了`CodecCommander`里面通常执行的也是`hda-verb 0x19 0x707 0x20`之类的语句, 只是在睡醒唤醒后`CodecCommander`已经无法执行`hda-verb 0x19 0x707 0x20`这样的语句了, 而`ALCPlugFix`却是以守护进程的方式存在于系统中, 无论什么时候它都会监听声卡的状态, 该切换时它就会发出命令切换, 这样就保证了声卡总是可以工作在正确的状态下.

基本上所有的耳机睡眠后没有声音的, 只需要使用一条命令:`hda-verb 0x19 0x707 0x20`, 就可以让你的耳机出声的. **当然前提是你得先加载了`CodecCommander`这个驱动**. 估计我写完这段你们看完了还是一头雾水, 我教大家一个简单的记法:`ALCPlugFix`作用于耳机, 所以你需要知道自己的两个耳机节点, 而`0x19`是耳麦的麦克风输入节点, 只要找对了耳麦的节点, 一条命令就可以搞定你的耳机切换.

### [](#杂音和底噪的去除)杂音和底噪的去除

在着手解决这个问题之前, 让我们先来搞清楚杂音和底噪产生的原因.

打开终端, 输入命令:

```
node_dump.sh
```

它会输出很多的信息出来, 我们只需要最后面的这组数据:

```
Configuration Default
	nid = 0x02 --> result 0x00000000
	nid = 0x03 --> result 0x00000000
	nid = 0x04 --> result 0x00000000
	nid = 0x05 --> result 0x00000000
	nid = 0x06 --> result 0x00000000
	nid = 0x07 --> result 0x00000000
	nid = 0x08 --> result 0x00000000
	nid = 0x09 --> result 0x00000000
	nid = 0x0a --> result 0x00000000
	nid = 0x0b --> result 0x00000000
	nid = 0x0c --> result 0x00000000
	nid = 0x0d --> result 0x00000000
	nid = 0x0e --> result 0x00000000
	nid = 0x0f --> result 0x00000000
	nid = 0x10 --> result 0x00000000
	nid = 0x11 --> result 0x00000000
	nid = 0x12 --> result 0x90a60170	# 内置麦克风
	nid = 0x13 --> result 0x400000f0
	nid = 0x14 --> result 0x90170140	# 喇叭
	nid = 0x15 --> result 0x00000000
	nid = 0x16 --> result 0x00000000
	nid = 0x17 --> result 0x00000000
	nid = 0x18 --> result 0x400000f0
	nid = 0x19 --> result 0x02ab1070	# 耳麦
	nid = 0x1a --> result 0x400000f0
	nid = 0x1b --> result 0x400000f0
	nid = 0x1c --> result 0x00000000
	nid = 0x1d --> result 0x400000f0
	nid = 0x1e --> result 0x400000f0
	nid = 0x1f --> result 0x00000000
	nid = 0x20 --> result 0x00000000
	nid = 0x21 --> result 0x022b1030	# 耳机
	nid = 0x22 --> result 0x00000000
	nid = 0x23 --> result 0x00000000
	nid = 0x24 --> result 0x00000000
```

这些数据里:`0x00000000`是无效的节点,`0x400000f0`是被屏蔽掉的节点. 其它的四个有效的节点我做了备注, 方便你们看清楚. 这些`PinConfigurations`决定了你的声卡音质. 如果你的命令输出信息里面除了有效节点和`0x400000f0`之外的其它任何格式的数据都会产生杂音和底噪. 比如`0x40000000`或者`0x411111f0`, 这些无效的节点就会一直在起作用, 这就好比一个流行乐队里, 需要一个架子鼓, 一台电钢琴, 两个吉他手, 再加上一个歌手, 如果你生硬地再加进一把小提琴, 再弄个竖琴, 就会不和谐, 就会产生噪音.

再让我们搬出`widget_dump.sh`命令:

```
widget_dump.sh
```

输出信息如下:

```
Pin Widget Control
	nid = 0x02 --> result 0x00000000
	nid = 0x03 --> result 0x00000000
	nid = 0x04 --> result 0x00000000
	nid = 0x05 --> result 0x00000000
	nid = 0x06 --> result 0x00000000
	nid = 0x07 --> result 0x00000000
	nid = 0x08 --> result 0x00000000
	nid = 0x09 --> result 0x00000000
	nid = 0x0a --> result 0x00000000
	nid = 0x0b --> result 0x00000000
	nid = 0x0c --> result 0x00000000
	nid = 0x0d --> result 0x00000000
	nid = 0x0e --> result 0x00000000
	nid = 0x0f --> result 0x00000000
	nid = 0x10 --> result 0x00000000
	nid = 0x11 --> result 0x00000000
	nid = 0x12 --> result 0x00000000	# 内置麦克风
	nid = 0x13 --> result 0x00000000
	nid = 0x14 --> result 0x00000000	# 喇叭	
	nid = 0x15 --> result 0x00000000
	nid = 0x16 --> result 0x00000000
	nid = 0x17 --> result 0x00000000
	nid = 0x18 --> result 0x00000000
	nid = 0x19 --> result 0x00000020	# 耳麦
	nid = 0x1a --> result 0x00000020				# 无效节点
	nid = 0x1b --> result 0x00000000
	nid = 0x1c --> result 0x00000000
	nid = 0x1d --> result 0x00000020				# 无效节点
	nid = 0x1e --> result 0x00000000
	nid = 0x1f --> result 0x00000000
	nid = 0x20 --> result 0x00000000
	nid = 0x21 --> result 0x000000c0	# 耳机
	nid = 0x22 --> result 0x00000000
	nid = 0x23 --> result 0x00000000
	nid = 0x24 --> result 0x00000000
```

我故意在四个有效的节点之外添加了两个无效的节点, 为了让大家看清楚一些, 这些无效的节点就会工作, 它会产生底噪, 尤其是戴上耳机, 关闭音源, 就会听到沙沙沙的声音, 也有人说它是电流声, 严重时还会产生杂音, 包括单声道输出等等.

既然我们知道了问题是如何产生的, 那么就让我们来消除这些杂音, 还原干净的声音.

让我们回到`整理ConfigData`这一章, **将有效节点之外的其它节点全部使用`f0 00 00 40`进行屏蔽**.

[](#声卡驱动了音量调节的图标也显示正常就是不发声怎么办)声卡驱动了, 音量调节的图标也显示正常, 就是不发声怎么办
-----------------------------------------------------------

如果您的声卡使用了 AppleALC, 也注入了正确的 ID 后不发声怎么办呢? 这个问题就出现在有效节点和路径不正确上面.  
所谓条条大路通罗马, 我们总不能在同一棵树上吊死吧. 现在可以请出 [VoodooHDA[教程]](https://github.com/daliansky/VoodooHDA-2.9.0-Clover) 万能声卡驱动程序[下载链接](https://github.com/daliansky/VoodooHDA-2.9.0-Clover/releases/download/V12/VoodooHDA.2.9.0.Clover-V12.pkg)了, 说它万能是不正确的, 如果它万能了估计就没`AppleALC`啥事儿了吧.

### [](#voodoohda-clover版安装教程)VoodooHDA Clover 版安装教程

你需要有现成的 Clover，它只会安装 VoodooHDA 和其他必需的组件来正常运行。

#### [](#方法一通过clover-uefiesp驱动)方法一: 通过 Clover UEFI/ESP 驱动

请点击下载:[VoodooHDA 2.9.0 Clover-V12.dmg](https://github.com/daliansky/VoodooHDA-2.9.0-Clover/releases/download/V12/VoodooHDA.2.9.0.Clover-V12.pkg)  
这个安装程序会自动安装到`/ESP/EFI/CLOVER/kexts/`以及下面的目录里 10.14/10.13/10.12/10.11/10.10/10.9/10.8/10.7/10.6  
安装目录取决于你安装的 macOS 系统决定.  
** 这个驱动将通过 Clover 加载而不需要安装到`/System/Library/Extensions/`, 同时它也不会删除系统自带的`AppleHDA.kext`, 您甚至无需备份`/Others/AppleALC.kext**`

#### [](#方法二clover传统模式)方法二: Clover 传统模式

你也可以安装 VoodooHDA.kext 内核扩展程序到 / EFI/Clover/kexts/10.14/10.1310.12/10.11/10.10/10.9/10.8/10.7/10.6  
这个驱动将通过 Clover 加载.

#### [](#方法三-经典方法)方法三: 经典方法

您也可以选择 10.6 到 10.12 的 Clasic 方法，这将在系统库扩展上安装 VoodooHDA.kext + AppleHDADisabler.kext

这将在应用程序上安装 VoodooHdaSettingsLoader.app，在 Library / PreferencePanes 上安装 VoodooHDA.prefPane  
在这两种方法中，在`usr/local/bin/getdump`上安装 getdump。

注意：对于所有 OS X 系统，VoodooHDA.prefPane 安装在 Library / PreferancesPanes 中

#### [](#视频教程请点击下方)视频教程请点击下方

```
⟱
```

[[![](https://i95.servimg.com/u/f95/18/50/18/69/video_10.png)](https://i95.servimg.com/u/f95/18/50/18/69/video_10.png)

### [](#getdump用法)getdump 用法

这里可以借助`VoodooHDA`自带的应用程序:[getdump](https://github.com/daliansky/VoodooHDA-2.9.0-Clover-V10/raw/master/getdump), 使用方法: 打开终端, 输入命令:

```
sudo cp ~/Downloads/getdump /usr/local/bin  # 将getdump命令复制到/usr/local/bin目录下
sudo chmod +x /usr/local/bin/getdump    	# 为getdump添加执行权限
getdump > ~/Desktop/voodoo_dump.txt
```

用它可以生成一份`voodoo_dump.txt`的文件, 里面会有两段以`DUMPING Playback/Record Paths`开头的文字描述, 这里面即包括了有效节点, 同时也包括了有效的路径  
[![](http://7.daliansky.net/path1.png)](http://7.daliansky.net/path1.png)  
[![](http://7.daliansky.net/path2.png)](http://7.daliansky.net/path2.png)  
从上面的两张截图中, 可以将有效的节点和路径整理出下面的表格:

<table><thead><tr><th>有效节点</th><th>10 进制</th><th>设备名称</th><th>路径</th></tr></thead><tbody><tr><td>0x14</td><td>20</td><td>Speaker out</td><td><strong>20-&gt;12-&gt;2</strong></td></tr><tr><td>0x21</td><td>33</td><td>HP out</td><td><strong>33-&gt;13-&gt;3</strong></td></tr><tr><td>0x19</td><td>25</td><td>Headset Mic in</td><td><strong>8-&gt;35-&gt;24</strong> <strong>8-&gt;35-&gt;29 备用</strong></td></tr><tr><td>0x12</td><td>18</td><td>Internal Mic in</td><td><strong>9-&gt;34-&gt;18</strong> <strong>9-&gt;34-&gt;29 备用</strong></td></tr></tbody></table>

有了这个表格, 再结合`configdata`, 找出有效的 ID, 编译`AppleALC`, 注入 ID 并使用声卡驱动.

[](#如果还不出声怎么办)如果还不出声怎么办？
------------------------

试试打开终端，输入这两条命令：

```
sudo kextunload /System/Library/Extensions/AppleHDA.kext
sudo kextload /System/Library/Extensions/AppleHDA.kext
```

**这两条命令是重新加载 AppleHDA 驱动**

制作声卡仿冒需要用到的工具, 我已经同步到仓库, 您可以打包下载也可以通过 git 同步到本地

*   下载链接:[https://github.com/daliansky/dell7000/tree/master/hda-tools](https://github.com/daliansky/dell7000/tree/master/hda-tools)
    
*   如何安装
    
    *   将`hda-tools`目录下的所有文件复制到`/usr/local/bin`下, 如果`/usr/local/bin`目录不存在, 需要新建立, 命令如下:
    
    ```
    sudo mkdir /usr/local/bin   # 建立/usr/local/bin子目录
    sudo cp ~/Downloads/hda-tools/* /usr/local/bin  # 将hda-tools目录下所有的应用程序复制到/usr/local/bin目录下
    ```
    
*   如何使用
    
    *   打开终端, 输入命令, 直接执行. 例如:
    
    ```
    verbit.sh codec#0 > codec_dump.txt    # 有效节点格式化
    codecgraph codec#0    # 自动生成pathmap图,文件位于codec#0相同目录下
    hda-verb 0x19 0x707 0x20    # 耳机睡眠唤醒无声修正
    widget_dump.sh  # 读取SET_PIN_WIDGET_CONTROL数值
    ```
    

> 未完，待续 [别问我何时续更]

[](#感谢名单)感谢名单
-------------

*   [Apple](https://www.apple.com/) 的 macOS
*   [RehabMan](https://github.com/rehabman) 维护的项目：[OS-X-Clover-Laptop-Config](https://github.com/RehabMan/OS-X-Clover-Laptop-Config) [Laptop-DSDT-Patch](https://github.com/RehabMan/Laptop-DSDT-Patch) [OS-X-USB-Inject-All](https://github.com/RehabMan/OS-X-USB-Inject-All) 等
*   [Acidanthera](https://github.com/acidanthera) 维护的项目：[OpenCorePkg](https://github.com/acidanthera/OpenCorePkg) [lilu](https://github.com/acidanthera/Lilu) [AirportBrcmFixup](https://github.com/acidanthera/AirportBrcmFixup) [WhateverGreen](https://github.com/acidanthera/WhateverGreen) [VirtualSMC](https://github.com/acidanthera/VirtualSMC) [AppleALC](https://github.com/acidanthera/AppleALC) [BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM) [MaciASL](https://github.com/acidanthera/MaciASL) 等
*   [headkaze](https://www.insanelymac.com/forum/profile/1364628-headkaze/) 提供的工具：[hackintool](https://github.com/headkaze/Hackintool) [PinConfigurator](https://github.com/headkaze/PinConfigurator) [BrcmPatchRAM](https://www.insanelymac.com/forum/topic/339175-brcmpatchram2-for-1015-catalina-broadcom-bluetooth-firmware-upload/)
*   [CloverHackyColor](https://github.com/CloverHackyColor) 维护的项目：[CloverBootloader](https://github.com/CloverHackyColor/CloverBootloader) [CloverThemes](https://github.com/CloverHackyColor/CloverThemes)
*   宪武整理的：[P-little](https://github.com/daliansky/P-little) [OC-little](https://github.com/daliansky/OC-little)
*   [chris1111](https://github.com/chris1111) 维护的项目：[VoodooHDA](https://github.com/chris1111/VoodooHDA-2.9.2-Clover-V15) [Wireless USB Adapter Clover](https://github.com/chris1111/Wireless-USB-Adapter-Clover)
*   [zxystd](https://github.com/zxystd) 开发的 [itlwm](https://github.com/zxystd/itlwm) [IntelBluetoothFirmware](https://github.com/zxystd/IntelBluetoothFirmware)
*   [lihaoyun6](https://github.com/lihaoyun6) 提供的工具：[CPU-S](https://github.com/lihaoyun6/CPU-S) [macOS-Displays-icon](https://github.com/lihaoyun6/macOS-Displays-icon) [SidecarPatcher](https://github.com/lihaoyun6/SidecarPatcher)
*   [sukka](https://github.com/SukkaW) 更新维护的[从 Clover 到 OpenCore —— Clover 迁移 OpenCore 指南](https://blog.skk.moe/post/from-clover-to-opencore/)
*   [xzhih](https://github.com/xzhih) 提供的工具：[one-key-hidpi](https://github.com/xzhih/one-key-hidpi)
*   [Bat.bat](https://github.com/williambj1) 更新维护的[精解 OpenCore](https://blog.daliansky.net/OpenCore-BootLoader.html)
*   [shuiyunxc](https://github.com/shuiyunxc) 更新维护的 [OpenCore 配置错误、故障与解决办法](https://shuiyunxc.gitee.io/2020/04/06/Faults/index/)
*   [athlonreg](https://github.com/athlonreg) 更新维护的 [OpenCore 0.5+ 部件补丁](https://blog.cloudops.ml/ocbook/) [Common-patches-for-hackintosh](https://github.com/athlonreg/Common-patches-for-hackintosh)
*   [github.com](https://blog.daliansky.net/github.com)
*   [码云 gitee.io](https://blog.daliansky.net/gitee.io)
*   [扣钉 coding.net](https://blog.daliansky.net/coding.net)

[](#参考及引用)参考及引用：
----------------

*   [https://deviwiki.com/wiki/Dell](https://deviwiki.com/wiki/Dell)
*   [https://deviwiki.com/wiki/Dell_Wireless_1820A_(DW1820A)](https://deviwiki.com/wiki/Dell_Wireless_1820A_(DW1820A))
*   [Hervé](https://blog.daliansky.net/%5Bhttps://osxlatitude.com/profile/4953-herv%C3%A9/%5D(https://osxlatitude.com/profile/4953-herv%C3%A9/)) 更新的 Broadcom 4350:[https://osxlatitude.com/forums/topic/12169-bcm4350-cards-registry-of-cardslaptops-interop/](https://osxlatitude.com/forums/topic/12169-bcm4350-cards-registry-of-cardslaptops-interop/)
*   [Hervé](https://blog.daliansky.net/%5Bhttps://osxlatitude.com/profile/4953-herv%C3%A9/%5D(https://osxlatitude.com/profile/4953-herv%C3%A9/)) 更新的 DW1820A 支持机型列表:[https://osxlatitude.com/forums/topic/11322-broadcom-bcm4350-cards-under-high-sierramojave/](https://osxlatitude.com/forums/topic/11322-broadcom-bcm4350-cards-under-high-sierramojave/)
*   [nickhx](https://osxlatitude.com/profile/129953-nickhx/) 提供的蓝牙驱动：[https://osxlatitude.com/forums/topic/11540-dw1820a-for-7490-help/?do=findComment&comment=92833](https://osxlatitude.com/forums/topic/11540-dw1820a-for-7490-help/?do=findComment&comment=92833)
*   [xjn819](https://blog.xjn819.com/)： [使用 OpenCore 引导黑苹果](https://blog.xjn819.com/?p=543) [300 系列主板正确使用 AptioMemoryFix.efi 的姿势 (重写版）](https://blog.xjn819.com/?p=317)
*   [insanelymac.com](https://www.insanelymac.com/)
*   [tonymacx86.com](https://www.tonymacx86.com/)
*   [远景论坛](http://bbs.pcbeta.com/)
*   [applelife.ru](https://applelife.ru/)
*   [olarila.com](https://www.olarila.com/)