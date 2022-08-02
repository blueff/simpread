> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/chenhuanqiangnihao/article/details/125652772)

[1、BUCK 和 BOOST 电路详解_541 板哥的博客 - CSDN 博客_buck 电路和 boost 电路](https://blog.csdn.net/u011816009/article/details/108439110?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_utm_term~default-1-108439110-blog-122600125.pc_relevant_vip_default&spm=1001.2101.3001.4242.1&utm_relevant_index=4 "1、BUCK和BOOST电路详解_541板哥的博客-CSDN博客_buck电路和boost电路")

[2、升压电路 (BOOST) 与降压电路(BUCK)_FFT 变换的博客 - CSDN 博客_buckboost 升降压电路原理](https://blog.csdn.net/lishen05/article/details/115759277?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-2-115759277-blog-108439110.pc_relevant_vip_default&spm=1001.2101.3001.4242.2&utm_relevant_index=5 "2、升压电路(BOOST)与降压电路(BUCK)_FFT变换的博客-CSDN博客_buckboost升降压电路原理") 

[3、BUCK/BOOST 电路原理分析_rayufo 的博客 - CSDN 博客_buckboost 电路原理](https://blog.csdn.net/rayufo/article/details/86678991?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2~default~BlogCommendFromBaidu~default-1-86678991-blog-108439110.pc_relevant_vip_default&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2~default~BlogCommendFromBaidu~default-1-86678991-blog-108439110.pc_relevant_vip_default&utm_relevant_index=1 "3、BUCK/BOOST电路原理分析_rayufo的博客-CSDN博客_buckboost电路原理") 

[4、深度解读四开关 Buck-Boost 电路及变换器 20191017_八皇后 OS 的博客 - CSDN 博客_buck-boost](https://csapp.blog.csdn.net/article/details/102616266?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~default-1-102616266-blog-108439110.pc_relevant_vip_default&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~BlogCommendFromBaidu~default-1-102616266-blog-108439110.pc_relevant_vip_default&utm_relevant_index=2 "4、深度解读四开关Buck-Boost电路及变换器 20191017_八皇后OS的博客-CSDN博客_buck-boost")

[5、硬件工程师笔试面试必考题】BUCK 电源工作原理及电源纹波分析 - 知乎](https://zhuanlan.zhihu.com/p/471588406 "5、硬件工程师笔试面试必考题】BUCK电源工作原理及电源纹波分析 - 知乎") 

（来自：[电子硬件攻城狮 - 知乎](https://www.zhihu.com/people/lu-shi-qiang-21 "电子硬件攻城狮 - 知乎")）

**目录**

[0. 前言](#t0)

[1.DC-DC 之 BUCK 电源](#t1)

[1.1 BUCK 电源关键器件及其作用](#t2)

[1.2 BUCK 电源工作原理](#t3)

[1.3 DC-DC 的 PCB 设计布局布线注意事项](#t4)

[1.4 DC-DC 与 LDO 的优缺点及选型](#t5)

[1.5 同等条件下 DC-DC 和 LDO 谁的体积更大？根本原因是什么？](#t6)

[2. 电源纹波产生的根本原因](#t7)

[3. 电源纹波的测试方法](#t8)

[4. 电源纹波的常见抑制方法](#t9)

[5. 结语及拓展](#t10)

[拓展 1：BOOST 电源](#t11)

[拓展 2：BUCK-BOOST 电源](#t12)

0. 前言
-----

DC-DC 电源知识是硬件工程师笔试面试中必考内容，本文详细讲解 DC-DC 的基本拓扑结构之 BUCK 电源。**干货较多，建议收藏慢慢品读！**

**以下问题都是硬件工程师笔试面试高频问题：**

*   BUCK 电源电路的基本结构是什么？[**_要求会画 BUCK 电路图_**]
*   BUCK 电源的原理是什么？[**_要求懂得其降压原理_**]
*   电源纹波产生的根本原因是什么？[**_要求掌握 DC-DC 电源的电源纹波根本来源_**]
*   抑制电源纹波的几种常用方法有哪些？[**_要求了解电源纹波的抑制方法_**]
*   电源纹波的测量方法和注意事项有哪些？[**_要求会测量电源纹波_**]
*   DC-DC 电路的 PCB 布局布线注意事项有哪些？[**_要求了解 DC-DC 电路的 PCB 布局_**]
*   DC-DC 与 LDO 各自的优缺点及应用场合是什么？[**_会根据实际需求选型 DC-DC 与 LDO_**]

**1.DC-DC 之 BUCK 电源**
---------------------

DC-DC 电源，即直流 - 直流变换器，是指将一个固定的[直流电压](https://link.zhihu.com/?target=https%3A//baike.baidu.com/item/%25E7%259B%25B4%25E6%25B5%2581%25E7%2594%25B5%25E5%258E%258B/8640960 "直流电压")变换为可变的直流电压，也称为直流斩波器。DC-DC 有多种拓扑结构，如 BUCK（降压）、BOOST（升压）、BUCK-BOOST（升降压）三大基本拓扑结构。其中 BUCK 电源即降压电源，实现降压输出，BUCK 电源电路基本结构如下图所示：

![](https://img-blog.csdnimg.cn/img_convert/6620ff667b925b1db6229d2589617e30.jpeg)

BUCK 电源电路基本结构

### **1.1 BUCK 电源关键器件及其作用**

关键器件有开关管 S、电感 L、电容 C、二极管 D。

*   **开关管 S：**可以导通和关断电流，常见的开关管有三极管、MOSFET 等。
*   **电感 L：**可以将电能转换成磁能储存起来，也能将磁能转换为电能再次释放。**需要注意：**电感在储能和释能转换时，电感的正负极会发生反向。流经电感的电流不能突变，只能逐步变大或变小。
*   **电容 C：**具有充放电功能，电容器两端电压高于外部电路电压时放电，反之充电。电容充放电不会发生正负极的反向。
*   **二极管 D：**具有单向导电性，电流只能单向流过。在 BUCK 电路中，二极管 D 形成了续流回路，因此 D 也叫作续流二极管。

### 1.2 BUCK 电源工作原理

一般通过 **PWM 波 “定频调宽” 或 PFM 波“定宽调频”** 两种方式来控制开关管的导通与关断。常用 PWM 波控制，本文所述为 PWM 控制开关管的通断。

PWM 波可用单片机产生，也可以使用 PWM 波专用芯片，如 UC3842。

![](https://img-blog.csdnimg.cn/img_convert/7d08c747c42ae589e5945e2bd656bf28.jpeg)

开关管 S 导通时电流回路

当 PWM 波控制开关管 S 导通时，图中红色回路为开关管导通时的回路。此时有下式成立：

![](https://img-blog.csdnimg.cn/img_convert/0c780780c246e8c1d7ae5461551dafb1.jpeg)

电流开始从左边的电源 Vin 正极流出，流向负极。电流流经续流二极管 D 的负极不能通过，继续前进流经电感 L，电感 L 将电能转换为磁能储存，电流继续前进流经电容 C，电容 C 充电，电流继续流过负载 R，回到电源 Vin 负极，整个电路通畅，输出端负载正常工作。

**此时电感 L 的工作状态：**此周期电感是左正右负，由于流过电感 L 的电流不能突变，所以负载 R 的电压是逐步增大的。

![](https://img-blog.csdnimg.cn/img_convert/b3dd90895980ae16bb2208b6a5439cd8.jpeg)

开关管 S 关断时电流回路

当 PWM 波控制开关管 S 关断时，图中红色回路为开关管关断时的回路。此时有下式成立：

![](https://img-blog.csdnimg.cn/img_convert/ad6d02940290e2b5861849fbed724ff3.jpeg)

当开关管 S 关断，电源 Vin 不再供电，电感 L 储存的磁能转换为电能释放，此时电感 L 的正负极反向（变成左负右正），电感 L 变成了电路里的电源。由于电流永远是从正极流向负极，所以形成了图中所示红色回路。此时续流二极管 D 正向导通，电感 L 释放的电流会逐步由大变小。

**电容 C 的作用：**当开关管 S 关断，电感 L 不能及时给负载 R 供电，此时电容 C 立马放电给负载 R 供电，电容 C 可以起到有效抑制电源纹波的作用。

![](https://img-blog.csdnimg.cn/img_convert/507afea51cc23a7305bbd63053aa0968.jpeg)

斩波实现降压公式推导 Vout=DVin

 从上式推导中可知  ，其中 D 为 PWM 波的占空比，占空比 D 的取值在 0~1 之间 ，因此实现了降压，这就是斩波实现降压的原理，可以通过改变 PWM 波的占空比 D 来控制降压的大小，如占空比为 50%，则  ，即实现了输出比输入小一半，也就是降压了一半。

### **1.3 DC-DC 的 PCB 设计布局布线注意事项**

*   DC-DC 的功率管脚应大面积铺铜皮较少电源的温升
*   根据载流原则处理好输入输出主回路（注意铺铜的通流能力和过孔的数量）
*   反馈线不能绕着电感 L 走
*   开关电源芯片及其电感下面尽量不要布其他信号线
*   二极管续流回路尽可能短
*   DC-DC 芯片的散热焊盘上需打矩阵过孔作进一步散热处理
*   布局要紧凑，输入输出主干道采用 “一” 字型获 “L” 型布局方式
*   关键滤波电容的放置要合理，采用先大后小的布局原则
*   输出供电电源应从输出电容取电
*   对于多路输出的开关电源尽量使相邻的电感垂直分布（麦克斯韦定律磁场相互垂直可抵消干扰）

### **1.4 DC-DC 与 LDO 的优缺点及选型**

*   DC-DC 优点：效率高，输入电压范围较宽，驱动力较强。
*   DC-DC 缺点：负载响应比 LDO 差，输出纹波比 LDO 大。
*   LDO 优点：负载响应快，稳定性好，输出纹波小。
*   LDO 缺点：效率低，输入输出的电压差不能太大，驱动能力较小，负载不能太大。

在实际使用中，应综合考量上述优缺点来进行器件选型。

### **1.5 同等条件下 DC-DC 和 LDO 谁的体积更大？根本原因是什么？**

同等条件下，LDO 的体积更大。由于 LDO 的转换率更低，耗散功率更大，所以发热量更大，因此在设计时会考虑散热问题，所以其体积在其他条件相同时会相对 DC-DC 要大一些。

**2. 电源纹波产生的根本原因**
------------------

随着开关管 S 的开关，电感 L 中的电流也是在输出电流的有效值上下波动的。所以在输出端也会出现一个与开关管 S 开关频率同频率的纹波，一般所说的纹波就是指这个，它与输出电容的容量和 ESR 有关系，纹波频率一般为几十到几百 kHz。

简单来说，一般所认为的纹波产生的**最根本原因**就是由于开关管 S 的通断过程导致电源产生波动，其频率等于开关管 S 的开关频率。

**3. 电源纹波的测试方法**
----------------

*   **示波器探头档位选定：**一般选定 X1 档；
*   **示波器通道耦合方式设置：**电源纹波是交流信号，因此设置 AC 耦合方式；
*   **示波器带宽限制设置：**一般设置为 20MHz，滤除电源的高频噪声成分，测量得到的纹波更加精准。
*   **接地方式：**需要就近接地，这样纹波测量才会更加精准，并且需要把接地夹换成接地环。
*   **示波器其他设置：**适当调整水平时基、垂直档位、和垂直偏移，让纹波合适的显示在示波器的中间位置。
*   **测量纹波：**可以使用示波器的 Measure 功能自动测量纹波峰峰值和频率。也可使用垂直光标手动测量纹波峰峰值和频率。

**4. 电源纹波的常见抑制方法**
------------------

*   **提高开关管的开关频率。**开关频率越高，纹波越小。
*   **加大电感和输出电容滤波。**根据开关电源的公式，电感内电流波动大小和电感值成反比，输出纹波和输出电容值成反比。所以加大电感值和输出电容值可以减小纹波。通常的做法，对于输出电容，使用铝电解电容以达到大容量的目的。但是电解电容在抑制高频噪声方面效果不是很好，而且 ESR 也比较大，所以会在它旁边并联一个陶瓷电容，来弥补铝电解电容的不足。
*   **二级滤波（即再加一级 LC 滤波器）。**LC 滤波器对纹波的抑制作用比较明显，根据要除去的纹波频率选择合适的电感电容构成 LC 滤波电路，一般能够很好的减小纹波。

电源纹波的抑制方法很多，但每种方法都有它自身的优缺点，所以在设计时需要综合考量。

5. 结语及拓展
--------

特别是应届毕业生求职时，硬件岗大概率（99%）会考核 DC-DC 相关知识，本期给大家分享的只是 DC-DC 三大基本拓扑结构中的 BUCK（降压）电源，关于 BUCK 电源，应届毕业生求职掌握本文内容就足够了。

此外 DC-DC 还有 BOOST 电源（升压）和 BUCK-BOOST 电源（升降压）两大拓扑结构。

本文暂且先把 BOOST 电源（升压）和 BUCK-BOOST 电源（升降压）两大拓扑结构的基本电路及原理公式给大家放在下方，供大家举一反三，学以致用。

### 拓展 1：BOOST 电源

BOOST 电源电路基本结构和原理公式如下：

![](https://img-blog.csdnimg.cn/img_convert/303089acceadb97a78c94c5d2c04e650.jpeg)

BOOST 电源电路基本结构

![](https://img-blog.csdnimg.cn/img_convert/ef3892c9bd54ca9b6b8ecc1ca9a11ff5.jpeg)

BOOST（升压）原理公式

### 拓展 2：BUCK-BOOST 电源

BUCK-BOOST 电源电路基本结构和原理公式如下：

![](https://img-blog.csdnimg.cn/img_convert/53c1274101f78d80bf557b1c2cec43b3.jpeg)

BUCK-BOOST（升降压）电源电路基本结构

![](https://img-blog.csdnimg.cn/img_convert/c1045d8a6f510fdaa38e03fe5ba0ba46.jpeg)

BUCK-BOOST（升降压）原理公式