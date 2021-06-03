> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [c.biancheng.net](http://c.biancheng.net/view/5319.html)

> 前面在介绍文件操作，一直在强调，打开的文件最后一定要关闭，否则会程序的运行造成意想不到的隐患。但是，即便使用 close() 做好了关闭文件的操作，如果在打开文件或文件操作过

在介绍 with as 语句时讲到，该语句操作的对象必须是上下文管理器。那么，到底什么是上下文管理器呢？

简单的理解，同时包含 __enter__() 和 __exit__() 方法的对象就是上下文管理器。也就是说，上下文管理器必须实现如下两个方法：

1.  __enter__(self)：进入上下文管理器自动调用的方法，该方法会在 with as 代码块执行之前执行。如果 with 语句有 as 子句，那么该方法的返回值会被赋值给 as 子句后的变量；该方法可以返回多个值，因此在 as 子句后面也可以指定多个变量（多个变量必须由 “()” 括起来组成元组）。
2.  __exit__（self, exc_type, exc_value, exc_traceback）：退出上下文管理器自动调用的方法。该方法会在 with as 代码块执行之后执行。如果 with as 代码块成功执行结束，程序自动调用该方法，调用该方法的三个参数都为 None：如果 with as 代码块因为异常而中止，程序也自动调用该方法，使用 sys.exc_info 得到的异常信息将作为调用该方法的参数。

当 with as 操作上下文管理器时，就会在执行语句体之前，先执行上下文管理器的 __enter__() 方法，然后再执行语句体，最后执行 __exit__() 方法。

构建上下文管理器，常见的有 2 种方式：基于类实现和基于生成器实现。

#### 基于类的上下文管理器

通过上面的介绍不难发现，只要一个类实现了 __enter__() 和 __exit__() 这 2 个方法，程序就可以使用 with as 语句来管理它，通过 __exit__() 方法的参数，即可判断出 with 代码块执行时是否遇到了异常。其实，上面程序中的文件对象也实现了这两个方法，因此可以接受 with as 语句的管理。

下面我们自定义一个实现上下文管理协议的类，并尝试用 with as 语句来管理它：

```
class FkResource:
    def __init__(self, tag):
        self.tag = tag
        print('构造器,初始化资源: %s' % tag)
    # 定义__enter__方法，with体之前的执行的方法
    def __enter__(self):
        print('[__enter__ %s]: ' % self.tag)
        # 该返回值将作为as子句中变量的值
        return 'fkit'  # 可以返回任意类型的值
    # 定义__exit__方法，with体之后的执行的方法
    def __exit__(self, exc_type, exc_value, exc_traceback):
        print('[__exit__ %s]: ' % self.tag)
        # exc_traceback为None，代表没有异常
        if exc_traceback is None:
            print('没有异常时关闭资源')
        else:
            print('遇到异常时关闭资源')
            return False   # 可以省略，默认返回None也被看做是False
with FkResource('孙悟空') as dr:
    print(dr)
    print('[with代码块] 没有异常')
print('------------------------------')
with FkResource('白骨精'):
    print('[with代码块] 异常之前的代码')
    raise Exception
    print('[with代码块] ~~~~~~~~异常之后的代码')
```

运行上面的程序，可以看到如下输出结果：

构造器, 初始化资源: 孙悟空  
[__enter__ 孙悟空]:  
fkit  
[with 代码块] 没有异常  
[__exit__ 孙悟空]:  
没有异常时关闭资源  
------------------------------  
构造器, 初始化资源: 白骨精  
[__enter__ 白骨精]:  
[with 代码块] 异常之前的代码  
[__exit__ 白骨精]:  
遇到异常时关闭资源  
Traceback (most recent call last):  
  File "C:\Users\mengma\Desktop\1.py", line 26, in <module>  
    raise Exception  
Exception

上面程序定义了一个 FkResource 类，并包含了 __enter__() 和 __exit__() 两个方法，因此该类的对象可以被 with as 语句管理。

此外，程序中两次使用 with as 语句管理 FkResource 对象。第一次代码块没有出现异常，第二次代码块出现了异常。从上面的输出结果来看，使用 with as 语句管理资源，无论代码块是否有异常，程序总可以自动执行 __exit__() 方法。

注意，当出现异常时，如果 __exit__ 返回 False（默认不写返回值时，即为 False），则会重新抛出异常，让 with as 之外的语句逻辑来处理异常；反之，如果返回 True，则忽略异常，不再对异常进行处理。

#### 基于生成器的上下文管理器

除了基于类的上下文管理器，它还可以基于生成器实现。接下来先看一个例子。比如，我们可以使用装饰器 contextlib.contextmanager，来定义自己所需的基于生成器的上下文管理器，用以支持 with as 语句：

```
from contextlib import contextmanager

@contextmanager
def file_manager(name, mode):
    try:
        f = open(name, mode)
        yield f
    finally:
        f.close()
       
with file_manager('a.txt', 'w') as f:
    f.write('hello world')
```

这段代码中，函数 file_manager() 就是一个生成器，当我们执行 with as 语句时，便会打开文件，并返回文件对象 f；当 with 语句执行完后，finally 中的关闭文件操作便会执行。另外可以看到，使用基于生成器的上下文管理器时，不再用定义 __enter__() 和 __exit__() 方法，但需要加上装饰器 @contextmanager，这一点新手很容易疏忽。

需要强调的是，基于类的上下文管理器和基于生成器的上下文管理器，这两者在功能上是一致的。只不过，基于类的上下文管理器更加灵活，适用于大型的系统开发，而基于生成器的上下文管理器更加方便、简洁，适用于中小型程序。但是，无论使用哪一种，不用忘记在方法 “__exit__()” 或者是 finally 块中释放资源，这一点尤其重要。