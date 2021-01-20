> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [ruleizhou.github.io](https://ruleizhou.github.io/2019/07/25/Android%20Treble%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/)

发表于 2019-07-25 更新于 2020-06-25 分类于 [Android](https://ruleizhou.github.io/categories/Android/)  

​ Android 8.0 版本的一项新元素是 Project Treble。这是 Android 操作系统框架在架构方面的一项重大改变，旨在让制造商以更低的成本更轻松、更快速地将设备更新到新版 Android 系统。Project Treble 适用于搭载 Android 8.0 及后续版本的所有新设备（这种新的架构已经在 Pixel 手机的开发者预览版中投入使用）。

1.1 系统更新
--------

[![](https://ruleizhou.github.io/2019/07/25/Android%20Treble%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/1-1-1.png)](https://ruleizhou.github.io/2019/07/25/Android%20Treble%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/1-1-1.png)

​ Android 7.x 及更早版本中没有正式的供应商接口，因此设备制造商必须更新大量 Android 代码才能将设备更新到新版 Android 系统：

[![](https://ruleizhou.github.io/2019/07/25/Android%20Treble%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/1-1-2.png)](https://ruleizhou.github.io/2019/07/25/Android%20Treble%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/1-1-2.png)

​ Treble 提供了一个稳定的新供应商接口，供设备制造商访问 Android 代码中特定于硬件的部分，这样一来，设备制造商只需更新 Android 操作系统框架，即可跳过芯片制造商直接提供新的 Android 版本。

1.2 Android 经典框架
----------------

为了更好的了解 Treble 架构里面的 HAL，首先了解一下 Android 的经典架构。

[![](https://ruleizhou.github.io/2019/07/25/Android%20Treble%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/1-2-1.png)](https://ruleizhou.github.io/2019/07/25/Android%20Treble%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/1-2-1.png)

​ 在 Android O 之前，HAL 是一个个的. so 库，通过 dlopen 来进行打开，库和 framework 位于同一个进程。如图所示

[![](https://ruleizhou.github.io/2019/07/25/Android%20Treble%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/1-2-2.bmp)](https://ruleizhou.github.io/2019/07/25/Android%20Treble%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/1-2-2.bmp)

1.3 Treble 架构
-------------

​ 为了能够让 Android O 之前的版本升级到 Android O，Android 设计了 Passthrough 模式，经过转换，可以方便的使用已经存在代码，不需要重新编写相关的 HAL。HIDL 分为两种模式：Passthrough 和 Binderized。

*   Binderized： Google 官方翻译成绑定试 HAL。
*   Passthrough：Google 官方翻译成直通式 HAL。

大致框架图如下，对于 Android O 之前的设备，对应图 1，对于从之前的设备升级到 O 的版本，对应图 2、图 3. 对于直接基于 Android O 开发的设备，对应图 4。

[![](https://ruleizhou.github.io/2019/07/25/Android%20Treble%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/1-3-1.png)](https://ruleizhou.github.io/2019/07/25/Android%20Treble%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/1-3-1.png)

这 ①②③④ 四种模式，是到目前为止四种实现架构。

*   ① 是 Treble Project 之前使用的实现架构，使用的是传统 HAL 和旧版 HAL
*   ② 直通模式，passthrough mode。如图所示，Framework 和 HAL 层工作在同一个进程当中，下面的 HAL 是使用 HIDL 封装后的库，是直通式 HAL。这些库文件也可用于 ③ 绑定模式
*   ③ 绑定模式，binderized mode。是直通式 HAL binder 化，变为绑定式 HAL。Framework 和 HAL 层工作在不同的进程，之间通过 Binder 进行 IPC
*   ④ 纯绑定式。相对于 ③ 来说，绑定式 HAL 中并不包含直通式 HAL，因此称为纯绑定式

新的架构之下，framework 和 hal 运行在不同的进程，所有的 HAL 采用新的 HIDL 技术来完成。

[![](https://ruleizhou.github.io/2019/07/25/Android%20Treble%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/1-3-2.bmp)](https://ruleizhou.github.io/2019/07/25/Android%20Treble%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/1-3-2.bmp)