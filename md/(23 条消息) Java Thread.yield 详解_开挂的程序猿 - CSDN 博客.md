> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/dabing69221/article/details/17426953)

前言：

   前几天复习了一下多线程，发现有许多网上讲的都很抽象，所以，自己把网上的一些案例总结了一下！

一. Thread.yield( ) 方法：

使当前线程从执行状态（运行状态）变为可执行态（就绪状态）。cpu 会从众多的可执行态里选择，也就是说，当前也就是刚刚的那个线程还是有可能会被再次执行到的，并不是说一定会执行其他线程而该线程在下一次中不会执行到了。

Java 线程中有一个 Thread.yield( ) 方法，很多人翻译成线程让步。顾名思义，就是说当一个线程使用了这个方法之后，它就会把自己 CPU 执行的时间让掉，让自己或者其它的线程运行。

打个比方：现在有很多人在排队上厕所，好不容易轮到这个人上厕所了，突然这个人说：“我要和大家来个竞赛，看谁先抢到厕所！”，然后所有的人在同一起跑线冲向厕所，有可能是别人抢到了，也有可能他自己有抢到了。我们还知道线程有个优先级的问题，那么手里有优先权的这些人就一定能抢到厕所的位置吗? 不一定的，他们只是概率上大些，也有可能没特权的抢到了。

例子：

```
package com.yield;
 
public class YieldTest extends Thread {
 
	public YieldTest(String name) {
		super(name);
	}
 
	@Override
	public void run() {
		for (int i = 1; i <= 50; i++) {
			System.out.println("" + this.getName() + "-----" + i);
			// 当i为30时，该线程就会把CPU时间让掉，让其他或者自己的线程执行（也就是谁先抢到谁执行）
			if (i == 30) {
				this.yield();
			}
		}
	}
 
	public static void main(String[] args) {
		YieldTest yt1 = new YieldTest("张三");
		YieldTest yt2 = new YieldTest("李四");
		yt1.start();
		yt2.start();
	}
}
```

运行结果：

第一种情况：李四（线程）当执行到 30 时会 CPU 时间让掉，这时张三（线程）抢到 CPU 时间并执行。

![](https://img-blog.csdn.net/20131219224322296?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGFiaW5nNjkyMjE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

第二种情况：李四（线程）当执行到 30 时会 CPU 时间让掉，这时李四（线程）抢到 CPU 时间并执行。

![](https://img-blog.csdn.net/20131219224720406?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZGFiaW5nNjkyMjE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)