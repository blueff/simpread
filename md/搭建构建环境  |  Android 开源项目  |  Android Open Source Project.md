> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [source.android.com](https://source.android.com/setup/build/initializing)

 .devsite-banner-announcement { background: #202124; color: white; font-weight: 500; } .devsite-banner-announcement a { color: white; font-weight: 700; } .devsite-banner-announcement a:link, .devsite-banner-announcement a:visited {background: 0;} .devsite-banner-announcement a:hover, .devsite-banner-announcement a:focus { text-decoration: none; } .devsite-banner-announcement .devsite-banner-message-text { margin: 0 auto; } 。

*   [AOSP](https://source.android.com/)
*   [设置](https://source.android.com/setup)
*   [开始](https://source.android.com/setup/start)

搭建构建环境
======

*   [目录](#top_of_page)
*   [选择分支](#choosing-a-branch)
*   [设置 Linux 构建环境](#setting-up-a-linux-build-environment)
    *   [安装所需的软件包 (Ubuntu 18.04)](#installing-required-packages-ubuntu-1804)
    *   [安装所需的软件包 (Ubuntu 14.04)](#installing-required-packages-ubuntu-1404)
    *   [安装所需的软件包 (Ubuntu 12.04)](#installing-required-packages-ubuntu-1204)
    *   [安装所需的软件包 (Ubuntu 10.04-11.10)](#installing-required-packages-ubuntu-1004-1110)
    *   [配置 USB 访问权限](#configuring-usb-access)
    *   [使用单独的输出目录](#using-a-separate-output-directory)
*   [设置 macOS 构建环境](#setting-up-a-mac-os-x-build-environment)
    *   [创建区分大小写的磁盘映像](#creating-a-case-sensitive-disk-image)
    *   [安装 Xcode 和其他软件包](#xcode-other-packages)
    *   [设置文件描述符数量上限](#setting-a-file-descriptor-limit)
*   [下一篇：下载源代码](#next-download-the-source)

本部分介绍了如何设置本地工作环境来构建 Android 源文件。您必须使用 Linux 或 macOS；目前不支持在 Windows 环境下进行构建。

如需简要了解整个代码审核和代码更新流程，请参阅[补丁程序的生命周期](/setup/contribute/life-of-a-patch)。

**注意**：本网站中所有命令的前面都带有美元符号 ($)，以便与输出或文件条目区分开来。您可以使用每个命令框右上角的 “点击复制” 功能来复制所有行（不包括美元符号），也可以三击每行来分别复制各行（不包括美元符号）。

选择分支
----

针对构建环境的某些要求是由您打算编译的源代码的版本决定的。如需查看可用分支的完整列表，请参阅 [Build 号](/setup/start/build-numbers)。您还可以选择下载并构建最新的源代码（称为 `master`）。如果您选择这么做，请在初始化代码库时直接忽略分支规范。

选择分支后，请按照下面的相应说明来设置构建环境。

设置 Linux 构建环境
-------------

以下说明适用于所有分支（包括 `master`）。

我们会定期在 Ubuntu LTS (14.04) 和 Debian 测试版本中对 Android 构建系统进行内部测试。其他大多数分发版本都应该具有所需的构建工具。

如果是 Gingerbread (2.3.x) 及更高版本（包括 `master` 分支），需要使用 64 位环境。如果是较低的版本，则可以在 32 位系统中进行编译。

**注意**：如需查看完整的硬件和软件要求列表，请参阅[要求](/setup/build/requirements)。然后，请按照下方适用于 Ubuntu 和 macOS 的详细说明进行操作。

### 安装所需的软件包 (Ubuntu 18.04)

您需要 64 位版本的 Ubuntu。

```
sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig


```

### 安装所需的软件包 (Ubuntu 14.04)

您需要 64 位版本的 Ubuntu（建议使用 Ubuntu 14.04）。

```
sudo apt-get install git-core gnupg flex bison gperf build-essential zip curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z-dev libgl1-mesa-dev libxml2-utils xsltproc unzip


```

**注意**：如需使用 SELinux 工具进行政策分析，您还需要安装 `python-networkx` 软件包。

**注意**：如果您使用的是 LDAP 并且希望运行 ART 主机测试，还需要安装 `libnss-sss:i386` 软件包。

### 安装所需的软件包 (Ubuntu 12.04)

您可以使用 Ubuntu 12.04 来构建较低版本的 Android。`master` 或最近推出的一些版本不支持 Ubuntu 12.04。

```
sudo apt-get install git gnupg flex bison gperf build-essential zip curl libc6-dev libncurses5-dev:i386 x11proto-core-dev libx11-dev:i386 libreadline6-dev:i386 libgl1-mesa-glx:i386 libgl1-mesa-dev g++-multilib mingw32 tofrodos python-markdown libxml2-utils xsltproc zlib1g-dev:i386
sudo ln -s /usr/lib/i386-linux-gnu/mesa/libGL.so.1 /usr/lib/i386-linux-gnu/libGL.so

```

### 安装所需的软件包 (Ubuntu 10.04-11.10)

不再支持在 Ubuntu 10.04-11.10 中进行构建，但它们仍可用来构建较低版本的 AOSP。

```
sudo apt-get install git gnupg flex bison gperf build-essential zip curl zlib1g-dev libc6-dev lib32ncurses5-dev ia32-libs x11proto-core-dev libx11-dev lib32readline5-dev lib32z-dev libgl1-mesa-dev g++-multilib mingw32 tofrodos python-markdown libxml2-utils xsltproc


```

在 Ubuntu 10.10 中，请运行以下命令：

```
sudo ln -s /usr/lib32/mesa/libGL.so.1 /usr/lib32/mesa/libGL.so


```

在 Ubuntu 11.10 中，请运行以下命令：

```
sudo apt-get install libx11-dev:i386


```

### 配置 USB 访问权限

按照说明[设置用于开发的设备](https://developer.android.com/studio/run/device.html#setting-up)，以安装适用于所有 Android 设备且由社区维护的默认 `udev` 规则集。

### 使用单独的输出目录

默认情况下，每次构建的输出都会存储在相应源代码树的 `out/` 子目录下。

在一些拥有多个存储设备的计算机上，如果将源文件和输出存储在单独的存储卷中，构建速度会更快。若要进一步提高构建速度，可以将输出存储在已针对速度（而非崩溃稳定性）进行优化的文件系统中，这是因为在文件系统损坏时可以重新生成所有文件。

如需进行这项设置，请导出 `OUT_DIR_COMMON_BASE` 变量，使其指向将存储输出目录的位置。

```
export OUT_DIR_COMMON_BASE=<path-to-your-out-directory>


```

对于每个单独的源代码树，其输出目录都将以其存放目录命名。例如，如果您有源代码树 `/source/master1` 和 `/source/master2`，且 `OUT_DIR_COMMON_BASE` 设置为 `/output`，则输出目录为 `/output/master1` 和 `/output/master2`。

避免将多个源代码树存储在具有相同名称的目录下，否则会导致输出目录共享终止，并且会出现不可预知的结果。仅 Jelly Bean (4.1) 及更高版本（包括 `master` 分支）支持这种做法。

设置 macOS 构建环境
-------------

在默认安装过程中，macOS 会在一个保留大小写但不区分大小写的文件系统中运行。Git 不支持这种类型的文件系统，而且此类文件系统会导致某些 Git 命令（如 `git status`）的行为出现异常。因此，我们建议您始终在区分大小写的文件系统中处理 AOSP 源文件。使用下文中介绍的磁盘映像可以非常轻松地做到这一点。

有了适当的文件系统，在新型 macOS 环境中构建 `master` 分支就会变得非常简单。要构建较早版本的分支，则需要一些额外的工具和 SDK。

### 创建区分大小写的磁盘映像

您可以使用磁盘映像在现有的 macOS 环境中创建区分大小写的文件系统。如需创建磁盘映像，请启动磁盘工具，然后选择**新建映像**。完成构建至少需要 25GB 空间；更大的空间能够更好地满足未来的需求。使用稀疏映像有助于节省空间，同时可以根据需要进行扩展。请选择 **Case sensitive, Journaled** 卷格式。

您也可以通过 shell 使用以下命令创建文件系统：

```
hdiutil create -type SPARSE -fs 'Case-sensitive Journaled HFS+' -size 250g ~/android.dmg.sparseimage


```

这将创建一个 `.dmg.sparseimage` 文件，该文件在装载后可用作具有 Android 开发所需格式的驱动盘。

如果您以后需要更大的存储卷，可以使用以下命令来调整稀疏映像的大小：

```
hdiutil resize -size <new-size-you-want>g ~/android.dmg.sparseimage


```

对于存储在主目录下的名为 `android.dmg.sparseimage` 的磁盘映像，您可以向 `~/.bash_profile` 中添加帮助程序函数：

*   如需在执行 `mountAndroid` 时装载映像，请运行以下命令：
    
    ```
    # mount the android file image
    mountAndroid() { hdiutil attach ~/android.dmg.sparseimage -mountpoint /Volumes/android; }
    
    ```
    
*   如需在执行 `umountAndroid` 时卸载映像，请运行以下命令：
    
    ```
    # unmount the android file image
    umountAndroid() { hdiutil detach /Volumes/android; }
    
    ```
    

装载 `android` 卷后，您将在其中开展所有工作。您可以像对待外接式驱动盘一样将其弹出（卸载）。

### 安装 Xcode 和其他软件包

1.  使用以下命令安装 Xcode 命令行工具：
    
    ```
    xcode-select --install
    
    ```
    
    对于旧版 macOS（10.8 或更低版本），您必须从 [Apple 开发者网站](http://developer.apple.com/)安装 Xcode。如果您尚未注册成为 Apple 开发者，则必须创建一个 Apple ID 才能下载。
2.  安装 [MacPorts](http://www.macports.org/install.php) 或 [Homebrew](https://brew.sh/) 以进行软件包管理。
3.  确保关联的目录位于 `~/.bash_profile` 文件的路径中：
    
    1.  MacPorts - `/opt/local/bin` 必须显示在 `/usr/bin` **之前**：
        
        ```
        export PATH=/opt/local/bin:$PATH
        
        ```
        
    2.  Homebrew - `/usr/local/bin`：
        
        ```
        export PATH=/usr/local/bin:$PATH
        
        ```
        
    3.  如果使用 MacPorts，请发出：
        
        ```
        POSIXLY_CORRECT=1 sudo port install git gnupg
        
        ```
        
    4.  如果使用 Homebrew，请发出：
        
        ```
        brew install git gnupg2
        
        ```
        
    
    ### 设置文件描述符数量上限
    
    在 macOS 中，可同时打开的文件描述符的默认数量上限太低，在高度并行的构建流程中，可能会超出此上限。如需提高此上限，请将下列行添加到 `~/.bash_profile` 中：
    
    ```
    # set the number of open files to be 1024
    ulimit -S -n 1024
    
    ```
    
    下一篇：下载源代码
    ---------
    
    构建环境已准备就绪！请继续阅读[下载源代码](/setup/build/downloading)一文。
    

此页内容对您有帮助吗？

此页内容对您有帮助吗？

此页内容对您有帮助吗？