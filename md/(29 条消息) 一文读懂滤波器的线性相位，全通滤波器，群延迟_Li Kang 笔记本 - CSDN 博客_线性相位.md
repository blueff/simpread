> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/s09094031/article/details/83755663)

### 一文读懂滤波器的线性相位，全通滤波器，群延迟

*   *   [1. 延迟](#1___4)
    *   [2. 全通滤波器](#2__16)
    *   [3. 相位延迟和群延迟](#3_35)
    *   [4. 实际生活中的例子](#4__52)
    *   [总结：](#_59)

数字信号处理最常见的面试题，请简述 FIR 和 IIR 的区别。其中的一个区别是 FIR 可以方便地实现线性相位。那这个线性相位指的是什么呢？本篇博客通过两个例子，

**延迟**

和

**全通滤波器**

，来解释这些概念。

先说结论：

**线性相位能保证信号中各频率成分的相对相位关系不改变。**

通俗解释：

**信号经过线性相位滤波器后，各个频率分量的延时时间是一样的**

。

1. 延迟
-----

举一个最简单的 FIR 的例子，延迟。假设 16kHz 的采样频率，一个采样周期的延迟，可以用 FIR 来表示。利用 Matlab 来观看这个滤波器的频率响应，代码如下。采样频率为 Fs = 16kHz， 采样周期为 Ts，Ts = 1/Fs。其中 num 是传递函数的分子，den 是传递函数的分母。分母只有 a0 = 1， 代表是一个 FIR 滤波器。分子 b0 = 0， b1 = 1, 代表是一个采样点的延迟。

```
num = [0,1]
den = [1,0]
fvtool(num,den)
```

下图中，蓝色的实线表示的幅频响应，为 0dB。红色实线表示相频响应，主要看相频响应。图中对 1k，2k，4kHz 频点的横纵坐标有截图，16kHz 采样率下，1kHz 的正弦信号一个完整的周期（这里说的周期指的是 2 π 2\pi 2π）内会得到 16 个采样值。一个采样周期的延迟，带来的相位变化是 − 2 π / 16 = − π / 8 = − 22.5 ° -2\pi/16=-\pi/8=-22.5\degree −2π/16=−π/8=−22.5°。而 2kHz 信号一个完整周期（ 2 π 2\pi 2π）内会得到 8 个采样值，那么一个采样周期的延迟带来的相位变化是 − 2 π / 8 = − π / 4 = − 45 ° -2\pi/8=-\pi/4=-45\degree −2π/8=−π/4=−45°，同理，对 4kHz 的信号，相位变化是 − 2 π / 4 = − π / 2 = − 90 ° -2\pi/4=-\pi/2=-90\degree −2π/4=−π/2=−90°。下图中也可以得到验证。  
![](https://img-blog.csdnimg.cn/20181110140618377.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3MwOTA5NDAzMQ==,size_16,color_FFFFFF,t_70)

这个滤波器对整个信号只产生了延迟一个采样点的效果，**信号的各个频率成分之间的相位关系没有改变**。从这个例子可以很清楚地看到，线性相位指的是滤波器对每个频点的相频响应是一个线性关系。如下图所示，上面的图表示 2kHz 信号和 4kHz 信号，中间的图表示两者之和，下面的图表示过了一个采样周期延迟的 FIR 滤波器。中间的图和下面的图，这两个频率成分的信号之间相位关系没有改变。  
![](https://img-blog.csdnimg.cn/20181110143141217.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3MwOTA5NDAzMQ==,size_16,color_FFFFFF,t_70)

2. 全通滤波器
--------

全通滤波器，幅频响应为 0db，但是可以改变个频率成分之间的相位关系。一阶全通滤波器有以下公式。  
A (z) = c + z − 1 1 + c z − 1 A(z) =\frac{c + z^{-1}}{1 + cz^{-1}} A(z)=1+cz−1c+z−1​  
c = t a n ( π ∗ f c / f s ) − 1 t a n ( π ∗ f c / f s ) + 1 c=\frac{tan(\pi*fc/fs) - 1}{tan(\pi*fc/fs) + 1} c=tan(π∗fc/fs)+1tan(π∗fc/fs)−1​其中 fc 为截止频率，fs 为采样频率。  
设计一个 fc = 2kHz，其幅频相频响应如下图所示。2kHz 处幅频响应为 − π / 2 -\pi/2 −π/2，4kHz 处幅频响应为 − 3 π / 4 -3\pi/4 −3π/4。很明显，这里已经不再是线性相位了。

```
fs = 16e3;
fc = 2e3;
c = (tan(pi*fc/fs) - 1)/(tan(pi*fc/fs) + 1)
num = [c,1];
den = [1,c];
fvtool(num,den)
```

![](https://img-blog.csdnimg.cn/20181110142713847.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3MwOTA5NDAzMQ==,size_16,color_FFFFFF,t_70)

我们再将 2kHz 和 4kHz 的和通过这个全通滤波器，看看输出是怎么样的。如下图所示，经过全通滤波器后，2kHz 和 4kHz 信号的相对相位已经改变了。2kHz 延迟了 π / 2 \pi/2 π/2，而 4kHz 延迟了 3 π / 4 3\pi/4 3π/4。最下面那幅图直接在输入信号上修改初始相位，也可以得到同样的效果。这里可以看出来，两个频率分量的相对相位关系发生了改变，两者之和的波形也发生了变化。  
![](https://img-blog.csdnimg.cn/20181110144749338.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3MwOTA5NDAzMQ==,size_16,color_FFFFFF,t_70)

3. 相位延迟和群延迟
-----------

以上两个例子可以看出，延迟和全通滤波器都能实现 0dB 的幅频响应，但相频响应区别很大。**相位延迟和群延迟**，就是用来描述这种不同。  
对于线性时不变系统，输入为 x (t) = e i w t x(t) = e^{iwt} x(t)=eiwt  
输出为 y (t) = H ( i w ) e i w t = ( ∣ H ( i w ) ∣ e i ϕ ( w ) ) e i w t = ∣ H ( i w ) ∣ e i ( ϕ ( w ) + w t ) y(t) = H(iw)e^{iwt} = (|H(iw)|e^{i\phi(w)})e^{iwt} = |H(iw)|e^{i(\phi(w)+wt)} y(t)=H(iw)eiwt=(∣H(iw)∣eiϕ(w))eiwt=∣H(iw)∣ei(ϕ(w)+wt)  
其中线性系统的相频响应为  
ϕ (w) = a r g { H ( i w ) } \phi(w) = arg\{H(iw)\} ϕ(w)=arg{H(iw)}  
群延迟和相位延迟可以表示为  
τ g = − d ϕ (w) d w \tau_g=-\frac{d\phi(w)}{dw} τg​=−dwdϕ(w)​  
τ ϕ = − ϕ (w) w \tau_\phi=-\frac{\phi(w)}{w} τϕ​=−wϕ(w)​  
这里先解释群延迟，群延迟是相位对频率的微分。若其是非常数，信号的各频率成分的相对相位关系将发生变化，从而产生相位失真。上面两个例子的群延迟分别如下图所示，纵坐标的单位为采样周期。

延迟（FIR）的群延迟就是 1 个采样周期的延迟，对于每一个频率分量都是一样的。  
![](https://img-blog.csdnimg.cn/20181117164723743.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3MwOTA5NDAzMQ==,size_16,color_FFFFFF,t_70)

全通滤波器（IIR）的群延迟会随频率变化，低频部分的延迟时间大于高频部分。信号经过这个系统之后，信号各频率成分的相对相位关系就改变了。  
![](https://img-blog.csdnimg.cn/20181117164741631.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3MwOTA5NDAzMQ==,size_16,color_FFFFFF,t_70)

4. 实际生活中的例子
-----------

就音乐厅来说，如果把舞台上音乐家的歌唱声或乐器发出的声音作为输入，听众听到的上述声音作为输出的话，那么音乐厅可以看成输入输出之间的一个系统。最理想的情况是，输出与输入之间只有一个类似于**延时**的线性相位滤波器，也即是舞台上唱什么歌，听众就能听到什么歌，只是时间上稍微有滞后。

如果音乐厅这个系统不是线性相位的，会出现什么情况呢？音乐是由很多不同的频率成分构成的。这时候音乐中有些频率成分很快就从舞台上传过来了，有些频率成分则要过一阵才传过来。这样组合起来的音乐，先不论是否悦耳，至少和舞台上的已经不一样了。这时候也就意味着坐在不同位置的听众，听到的将是不同的音乐。这是人们不希望看到的。

这种情况下，必须要求线性相位的响应。该举例转载于博客《[线性相位重要性的理解](https://blog.csdn.net/deepdsp/article/details/6804888)》。下面这篇博客介绍了线性滤波器在 ECG 信号中的使用，也是通俗易懂，可以参考。[《为什么设计的滤波器一定要用线性相位》](https://blog.csdn.net/u012947501/article/details/88670548)

总结：
---

1.  线性相位能保证信号中各频率成分的相对相位关系不改变。通俗解释是：信号经过滤波器后，各个频率分量的延时时间是一样的。
2.  延迟（FIR）和全通滤波器（IIR）都能对信号实现 0dB 的幅频响应。但是 FIR 能实现线性相位，而且群延迟为一个常数。
3.  要让信号传输不失真，这个传输系统必须具有**常数幅度增益**和**线性相位延迟**。线性时不变系统只能保证常数倍的幅度增益，并不能保证线性相位延迟。
4.  音乐厅的冲激响应必须要求线性相位。