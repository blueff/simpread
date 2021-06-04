> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/liubiao/p/6772873.html)

线程间通信
-----

#### 1.Queue

使用线程队列有一个要注意的问题是，向队列中添加数据项时并不会复制此数据项，线程间通信实际上是在线程间传递对象引用。如果你担心对象的共享状态，那你最好只传递不可修改的数据结构（如：整型、字符串或者元组）或者一个对象的深拷贝。

Queue 对象提供一些在当前上下文很有用的附加特性。比如在创建 Queue 对象时提供可选的 size 参数来限制可以添加到队列中的元素数量。对于 “生产者” 与“消费者”速度有差异的情况，为队列中的元素数量添加上限是有意义的。比如，一个 “生产者” 产生项目的速度比 “消费者”“消费” 的速度快，那么使用固定大小的队列就可以在队列已满的时候阻塞队列，以免未预期的连锁效应扩散整个程序造成死锁或者程序运行失常。在通信的线程之间进行 “流量控制” 是一个看起来容易实现起来困难的问题。如果你发现自己曾经试图通过摆弄队列大小来解决一个问题，这也许就标志着你的程序可能存在脆弱设计或者固有的可伸缩问题。 get() 和 put() 方法都支持非阻塞方式和设定超时。

```
import queue
q = queue.Queue()
try:
	data = q.get(block=False)
except queue.Empty:
...
try:
	q.put(item, block=False)
except queue.Full:
...
try:
	data = q.get(timeout=5.0)
except queue.Empty:
...
```

```
def producer(q):
	...
	try:
		q.put(item, block=False)
	except queue.Full:
		log.warning('queued item %r discarded!', item)
```

```
_running = True
def consumer(q):
	while _running:
	try:
		item = q.get(timeout=5.0)
		# Process item
		...
	except queue.Empty:
		pass
```

最后，有 q.qsize() ， q.full() ， q.empty() 等实用方法可以获取一个队列的当前大小和状态。但要注意，这些方法都不是线程安全的。可能你对一个队列使用 empty() 判断出这个队列为空，但同时另外一个线程可能已经向这个队列中插入一个数据项。所以，你最好不要在你的代码中使用这些方法。

为了避免出现死锁的情况，使用锁机制的程序应该设定为每个线程一次只允许获取一个锁。如果不能这样做的话，你就需要更高级的死锁避免机制。在 threading 库中还提供了其他的同步原语，比如 RLock 和 Semaphore 对象。

##### Queue 提供的方法：

```
task_done()
```

意味着之前入队的一个任务已经完成。由队列的消费者线程调用。每一个 get() 调用得到一个任务，接下来的 task_done() 调用告诉队列该任务已经处理完毕。

如果当前一个 join() 正在阻塞，它将在队列中的所有任务都处理完时恢复执行（即每一个由 put() 调用入队的任务都有一个对应的 task_done() 调用）。

```
join()
```

阻塞调用线程，直到队列中的所有任务被处理掉。

只要有数据被加入队列，未完成的任务数就会增加。当消费者线程调用 task_done()（意味着有消费者取得任务并完成任务），未完成的任务数就会减少。当未完成的任务数降到 0，join() 解除阻塞。

```
put(item[, block[, timeout]])
```

将 item 放入队列中。  
1. 如果可选的参数 block 为 True 且 timeout 为空对象（默认的情况，阻塞调用，无超时）。  
2. 如果 timeout 是个正整数，阻塞调用进程最多 timeout 秒，如果一直无空空间可用，抛出 Full 异常（带超时的阻塞调用）。  
3. 如果 block 为 False，如果有空闲空间可用将数据放入队列，否则立即抛出 Full 异常

其非阻塞版本为 put_nowait 等同于 put(item, False)

```
get([block[, timeout]])
```

从队列中移除并返回一个数据。block 跟 timeout 参数同 put 方法

其非阻塞方法为 get_nowait() 相当与 get(False)

```
empty()
```

如果队列为空，返回 True, 反之返回 False

同步机制
----

#### Event

线程的一个关键特性是每个线程都是独立运行且状态不可预测。如果程序中的其他线程需要通过断某个线程的状态来确定自己下一步的操作，这时线程同步问题就会变得非常棘手。为了解决这些问题，我们需要使用 threading 库中的 Event 对象。Event 对象包含一个可由线程设置的信号标志，它允许线程等待某些事件的发生。在初始情况下，event 对象中的信号标志被设置假。如果有线程等待一个 event 对象，而这个 event 对象的标志为假，那么这个线程将会被一直阻塞直至该标志为真。一个线程如果将一个 event 对象的信号标志设置为真，它将唤醒所有等待个 event 对象的线程。如果一个线程等待一个已经被设置为真的 event 对象，那么它将忽略这个事件，继续执行。

```
from threading import Thread, Event
import time

def countdown(n, start_evt):
    print('countdown is starting...')
    start_evt.set()
    while n > 0:
        print('T-minus', n)
        n -= 1
        time.sleep(5)

start_evt = Event()  # 可通过Event 判断线程的是否已运行
t = Thread(target=countdown, args=(10, start_evt))
t.start()

print('launching countdown...')
start_evt.wait()  # 等待countdown执行

# event 对象的一个重要特点是当它被设置为真时会唤醒所有等待它的线程

print('countdown is running...')
```

#### Semaphore（信号量）

在多线程编程中，为了防止不同的线程同时对一个公用的资源（比如全部变量）进行修改，需要进行同时访问的数量（通常是 1）的限制。信号量同步基于内部计数器，每调用一次 acquire()，计数器减 1；每调用一次 release()，计数器加 1. 当计数器为 0 时，acquire() 调用被阻塞。

```
from threading import Semaphore, Lock, RLock, Condition, Event, Thread
import time

# 信号量
sema = Semaphore(3)  #限制同时能访问资源的数量为3
  
def foo(tid):
    with sema:
        print('{} acquire sema'.format(tid))
        time.sleep(1)
    print('{} release sema'.format(tid))


threads = []

for i in range(5):
    t = Thread(target=foo, args=(i,))
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```

#### Lock（锁）

互斥锁为资源引入一个状态：锁定 / 非锁定。某个线程要更改共享数据时，先将其锁定，此时资源的状态为 “锁定”，其他线程不能更改；直到该线程释放资源，将资源的状态变成 “非锁定”，其他的线程才能再次锁定该资源。互斥锁保证了每次只有一个线程进行写入操作，从而保证了多线程情况下数据的正确性。

```
#创建锁
mutex = threading.Lock()
#锁定
mutex.acquire([timeout])
#释放
mutex.release()
```

#### RLock（可重入锁）

为了支持在同一线程中多次请求同一资源，python 提供了 “可重入锁”：threading.RLock。RLock 内部维护着一个 Lock 和一个 counter 变量，counter 记录了 acquire 的次数，从而使得资源可以被多次 acquire。直到一个线程所有的 acquire 都被 release，其他的线程才能获得资源。

```
import threading
import time

class MyThread(threading.Thread):
    def run(self):
        global num 
        time.sleep(1)

        if mutex.acquire(1):  
            num = num+1
            msg = self.name+' set num to '+str(num)
            print msg
            mutex.acquire()
            mutex.release()
            mutex.release()
num = 0
mutex = threading.RLock()
def test():
    for i in range(5):
        t = MyThread()
        t.start()
if __name__ == '__main__':
    test()
```

#### Condition（条件变量）

Condition 被称为条件变量，除了提供与 Lock 类似的 acquire 和 release 方法外，还提供了 wait 和 notify 方法。线程首先 acquire 一个条件变量，然后判断一些条件。如果条件不满足则 wait；如果条件满足，进行一些处理改变条件后，通过 notify 方法通知其他线程，其他处于 wait 状态的线程接到通知后会重新判断条件。不断的重复这一过程，从而解决复杂的同步问题。

可以认为 Condition 对象维护了一个锁（Lock/RLock) 和一个 waiting 池。线程通过 acquire 获得 Condition 对象，当调用 wait 方法时，线程会释放 Condition 内部的锁并进入 blocked 状态，同时在 waiting 池中记录这个线程。当调用 notify 方法时，Condition 对象会从 waiting 池中挑选一个线程，通知其调用 acquire 方法尝试取到锁。

Condition 对象的构造函数可以接受一个 Lock/RLock 对象作为参数，如果没有指定，则 Condition 对象会在内部自行创建一个 RLock。

除了 notify 方法外，Condition 对象还提供了 notifyAll 方法，可以通知 waiting 池中的所有线程尝试 acquire 内部锁。由于上述机制，处于 waiting 状态的线程只能通过 notify 方法唤醒，所以 notifyAll 的作用在于防止有线程永远处于沉默状态。

```
import threading
import time

class Producer:
    def run(self):
        global count
        while True:
            if con.acquire():
                if count > 1000:
                    con.wait()
                else:
                    count += 100
                    msg = threading.current_thread().name + ' produce 100, count=' + str(count)
                    print(msg)
                    con.notify()  # 通知 waiting线程池中的线程
                con.release()
                time.sleep(1)

count = 0
con = threading.Condition()

class Consumer:
    def run(self):
        global count
        while True:
            if con.acquire():
                if count < 100:
                    con.wait()
                else:
                    count -= 3
                    msg = threading.current_thread().name + ' consumer 3, count=' + str(count)
                    print(msg)
                    con.notify()
                con.release()
                time.sleep(3)

producer = Producer()
consumer = Consumer()
```