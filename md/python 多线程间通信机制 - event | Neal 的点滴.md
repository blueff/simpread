> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhaochj.github.io](https://zhaochj.github.io/2016/08/14/2016-08-14-%E7%BA%BF%E7%A8%8B%E7%9A%84event%E7%89%B9%E6%80%A7/)

> python 学习笔记之多线程间通信 - event

python 学习笔记之多线程间通信 - event

线程间的通讯 -- event
---------------

event 是多线程间通信的一种简单机制，一个线程发出 event 信号，其他线程等待这个信号。

以一个例子来说明，如下：

```
import threading
import logging
import time
 
logging.basicConfig(level=logging.DEBUG, format='%(asctime)s %(levelname)s [%(threadName)s] %(message)s')
 
 
def worker(event):
    while not event.is_set():
        logging.debug('in worker fun, event is set ? {0}'.format(event.is_set()))
    logging.debug('event is set')
 
 
def set(event):
    time.sleep(1)
    event.set()
    logging.debug('in set fun, event is set ? {0}'.format(event.is_set()))
 
 
if __name__ == '__main__':
    event = threading.Event()
    w = threading.Thread(target=worker, args=(event,), )
    w.start()
    s = threading.Thread(target=set, args=(event,), )
    s.start()
```

在上边的代码中定义了两个函数，并在测试代码上启用两个线程分别调用这两个函数，在调用前要初始化`event`对象。运行上边的代码会发现一直输出`2016-08-14 20:36:49,059 DEBUG [worker] in worker fun, event is set ? False`, 即是调用`worker`函数的线程在不断执行，1 秒后输出了`2016-08-14 20:36:49,063 DEBUG [set] in set fun, event is set ? True`，紧跟着输出`2016-08-14 20:36:49,063 DEBUG [worker] event is set`，如下：

```
2016-08-14 20:36:49,061 DEBUG [worker] in worker fun, event is set ? False
2016-08-14 20:36:49,061 DEBUG [worker] in worker fun, event is set ? False
2016-08-14 20:36:49,061 DEBUG [worker] in worker fun, event is set ? False
2016-08-14 20:36:49,061 DEBUG [worker] in worker fun, event is set ? False
2016-08-14 20:36:49,061 DEBUG [worker] in worker fun, event is set ? False
2016-08-14 20:36:49,062 DEBUG [worker] in worker fun, event is set ? False
2016-08-14 20:36:49,062 DEBUG [worker] in worker fun, event is set ? False
....略.....
2016-08-14 20:36:49,062 DEBUG [worker] in worker fun, event is set ? False
2016-08-14 20:36:49,062 DEBUG [worker] in worker fun, event is set ? False
2016-08-14 20:36:49,062 DEBUG [worker] in worker fun, event is set ? False
2016-08-14 20:36:49,062 DEBUG [worker] in worker fun, event is set ? False
2016-08-14 20:36:49,062 DEBUG [worker] in worker fun, event is set ? False
2016-08-14 20:36:49,062 DEBUG [worker] in worker fun, event is set ? False
2016-08-14 20:36:49,062 DEBUG [worker] in worker fun, event is set ? False
2016-08-14 20:36:49,063 DEBUG [set] in set fun, event is set ? True
2016-08-14 20:36:49,063 DEBUG [worker] event is set
```

为什么会这样？

因为在测试代码中启动了两个线程分别调用`worker`和`set`函数，`worker`函数判断当 event 没有被 set 时打印类似这样日志`in worker fun, event is set ? False`。而`set`函数负责将线程内部的标志设置为`true`，即执行`event.set()`语句，它将唤醒所有线程并告知`event`的内部标志已为`true`，`set`函数只是 sleep 了 1 秒钟，调用`woker`函数的线程就执行了很多次了。此时执行`worker函数`的线程收到这个信号后执行到`while not event.is_set()`语句时，`event.is_set()`返回了`true`，所以整个 while 语句返回`False`，那 while 循环中的语句将被跳过，而去执行`logging.debug('event is set')`。

*   event.wait()

wait 方法表示阻塞直到线程内部的标志为`true`，即有线程调用`set()`方法。以下边的例子：

```
import threading
import logging
import time
 
logging.basicConfig(level=logging.DEBUG, format='%(asctime)s %(levelname)s [%(threadName)s] %(message)s')
 
 
def worker(event):
    while not event.is_set():
        event.wait()
        logging.debug('in worker fun, event is set ? {0}'.format(event.is_set()))
    logging.debug('event is set')
 
 
def set(event):
    time.sleep(4)
    event.set()
    logging.debug('in set fun, event is set ? {0}'.format(event.is_set()))
 
 
if __name__ == '__main__':
    event = threading.Event()
    w = threading.Thread(target=worker, args=(event,), )
    w.start()
    s = threading.Thread(target=set, args=(event,), )
    s.start()
```

执行上边的代码时，`w`线程会被一直阻塞，直到 4 秒种之后，`set`函数执行到`event.set()`，内部标志被设置成为`true`，通知其他所有线程，`w`线程收到信号后不再阻塞。输出如下：

```
2016-08-14 21:45:26,326 DEBUG [set] in set fun, event is set ? True
2016-08-14 21:45:26,326 DEBUG [worker] in worker fun, event is set ? True
2016-08-14 21:45:26,326 DEBUG [worker] event is set
 
Process finished with exit code 0
```

wait 可以设置超时时间，表示只阻塞线程一定的时间，如果收到了`event.set()`信号，线程退出，否则线程被阻塞，只是不是一直阻塞，而是按照设定的时间阻塞线程，如下设置`wait`时间，代码如下：

```
import threading
import logging
import time
 
logging.basicConfig(level=logging.DEBUG, format='%(asctime)s %(levelname)s [%(threadName)s] %(message)s')
 
 
def worker(event):
    while not event.is_set():
        event.wait(timeout=1)
        logging.debug('in worker fun, event is set ? {0}'.format(event.is_set()))
    logging.debug('event is set')
 
 
def set(event):
    time.sleep(4)
    event.set()
    logging.debug('in set fun, event is set ? {0}'.format(event.is_set()))
 
 
if __name__ == '__main__':
    event = threading.Event()
    w = threading.Thread(target=worker, args=(event,), )
    w.start()
    s = threading.Thread(target=set, args=(event,), )
    s.start()
```

当执行上边代码时，每隔一秒执行`worker`函数，当到第 4 秒时，`event`对象被`set`，`w`线程收到信号就退出。