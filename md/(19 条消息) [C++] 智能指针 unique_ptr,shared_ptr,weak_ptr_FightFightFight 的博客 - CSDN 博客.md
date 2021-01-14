> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/FightFightFight/article/details/85252350)

C++ 中对于动态内存的使用非常严格，一次申请必须对应一次释放，否则将造成内存泄漏。这也要求程序员格外小心，比如如下示例:

```
void getStr() {
    std::string * pstr = new std::string();//pstr为局部变量
    *pstr = "Hello world";
    ....
    return;
}
```

当该方法执行完毕后，局部变量 pstr 所占用内存自动释放，但其指向的内存则一直驻留。必须通过`delete pstr`来释放，因此这里将造成内存泄漏。

还有一种情况就是当程序因异常而终止时:

```
#include <stdexcept>

void getStr() {
    char * pch = new char[1024];
    ...
    if (true)
        throw logic_error("throw a exception");
    delete [] pch;
}
```

虽然在这里没有忘记 delete 释放内存，但是当程序执行到 throw 时，将不再执行 throw 之后的语句，因此导致内存泄漏。针对于这个例子，虽然可以通过如下的方式避免这个问题，但依旧需要非常小心:

```
#include <stdexcept>

void getStr() {
    char * pch = new char[1024];
    ...
    if (true)
    try{
        throw logic_error("throw a exception");
    } catch(...){
        delete [] pch;
        throw;
    }
    delete [] pch;
}
```

为了让程序员不再因内存泄漏而操心，C++11 中提供了几个用于动态内存管理的智能指针。

智能指针的原理
=======

如果在指针对象过期时，让它的析构函数删除它指向的内存，那不就避免内存的泄漏了吗？比如当自动变量 pstr 被销毁时，让他的析构函数可以释放它指向的内。

智能指针正是采用这种方式实现的，在智能指针类中，定义了类似指针的对象，然后将`new`获得的地址赋给这个对象，当智能指针过期后，其析构函数将使用`delete`来释放这块内存。

使用智能指针
======

智能指针模板类位于头文件`memory`中，因此使用时必须包含该头文件。`memory`中提供了四种智能指针模板类:`auto_ptr`,`unique_ptr`,`shared_ptr`,`weak_ptr`, 其中`auto_ptr`在 C++11 中被弃用，在 C++17 中被移除，因此主要学习其余三种。

unique_ptr
----------

`unique_ptr`类模板如下:

```
template<class T, class Deleter = std::default_delete<T>> 
class unique_ptr;

template <class T,class Deleter> 
class unique_ptr<T[], Deleter>;
```

因此，`unique_ptr`有两个版本:

*   1. 管理个对象 (以 new 分配)
*   2. 管理动态分配的对象数组 (以 new[] 分配)

### 构造方法

`unique_ptr`的构造方法有多个，但常用如下几个个:

```
constexpr unique_ptr() noexcept;
constexpr unique_ptr (nullptr_t) noexcept : unique_ptr() {}
explicit unique_ptr(pointer p) noexcept;
```

`unique_ptr`通过建立所有权概念来管理对象，也就是说，对于特定的对象，只能有一个智能指针可以拥有它。因此:

*   1. 不能使用一个`unique_ptr`对象来初始化另一个`unique_ptr`对象，其拷贝构造不可用:`unique_ptr (const unique_ptr&) = delete`;
*   2. 不能使用一个`unique_ptr`对象来给另一个`unique_ptr`赋值，其赋值运算符不可用:`unique_ptr& operator= (const unique_ptr&) = delete`.

然而，当试图将一个`unique_ptr`对象赋给另一个`unique_ptr`时，如果源`unique_ptr`是一个临时右值，则编译器允许这样做，如:

```
std::unique_ptr<Person> get_uptr(Person* p) {
        std::unique_ptr<Person> temp (p);
        return temp;
}
std::unique_ptr<Person> uptr4 = get_uptr(new Person("XiaoWang"));
```

这是通过移动构造来区分的。

如果需要将一个`unique_ptr`对象赋给另一个`unique_ptr`，则使用`std::move()`函数;

### 成员函数

*   `get()`: 返回指向被管理对象的指针。
*   `reset()`: 替换被管理对象;
*   `release()`: 返回一个指向被管理对象的指针，并释放所有权.
*   `swap()`: 替换被管理对象;
*   `operator bool`: 检查是否有关联的被管理对象.

下面为使用`unique_ptr`示例:

```
#include <iostream>
#include <memory>
#include <string>

class Person { private: std::string name; public: Person(const std::string& name):name(name){ std::cout << "Person " << name << " created." << std::endl;
        } void show() const { std::cout << "Name: " << name << std::endl;
        } ~Person(){ std::cout << "Person " << name << " deleted." << std::endl;
        } }; int main()
{ // 1.创建unique_ptr管理Person*对象 Person* p = new Person("XiaoQiang"); std::unique_ptr<Person> uptr1; std::unique_ptr<Person> uptr2(new Person("Zhangsan")); // 2.不能用一个unique_ptr来初始化另一个unique_ptr
        //拷贝构造不可用:unique_ptr (const unique_ptr&) = delete
        //std::unique_ptr<Person> uptr3 = uptr2; // 3.同一对象只能被一个unique_ptr拥有，否则编译出错
        //赋值运算符不可用:operator=(const unique_ptr& ) = delete; //uptr1 = uptr2; // 3.get()返回管理对象的指针 Person* pt = uptr2.get(); pt->show(); // 4.reset()替换管理对象,原来对象将销毁 uptr2.reset(p); uptr2.get()->show(); // 5.release()返回管理对象，并释放所有权 uptr2.release()->show(); // 6.operator bool 判断是否有关联对象
        if(uptr2)
                uptr2->show(); else 
                std::cout << "uprt is null" << std::endl; // 7.swap()交换管理对象
        std::unique_ptr<Person> uptr3(new Person("LiSi")); std::unique_ptr<Person> uptr4(new Person("James")); uptr3.swap(uptr4); uptr3->show(); // 8.std::move()函数将一个unique_ptr对象赋给另一个unique_ptr对象 uptr1 = std::move(uptr4); uptr1->show(); // 9.c++14中，使用std::make_unique(T&& t)创建unique_ptr,参数为右值引用
        auto uptr5 = std::make_unique<Person>("LaoWang"); uptr5->show(); return 0;
} /*
@ubuntu:~$ g++ uptr.cpp -o uptr --std=c++14
@ubuntu:~$ ./uptr 
Person XiaoQiang created.
Person Zhangsan created.
Name: Zhangsan
Person Zhangsan deleted.
Name: XiaoQiang
Name: XiaoQiang
uprt is null
Person LiSi created.
Person James created.
Name: James
Name: LiSi
Person LaoWang created.
Name: LaoWang
Person LaoWang deleted.
Person James deleted.
Person LiSi deleted.
*/
```

shared_ptr
----------

`shared_ptr`通过引用计数机制来管理对象，多个 shared_ptr 对象可占有同一对象。可以通过成员函数`use_count()`返回`shared_ptr`所指对象的引用计数。它的类模板如下:

```
template <class T> class shared_ptr;
```

### 构造方法

其常用构造方法如下:

```
constexpr shared_ptr() noexcept;
constexpr shared_ptr(nullptr_t) : shared_ptr() {}
template <class U> explicit shared_ptr (U* p);
shared_ptr (const shared_ptr& x) noexcept;
template <class U> shared_ptr (const shared_ptr<U>& x) noexcept;
template <class U> explicit shared_ptr (const weak_ptr<U>& x);
template <class Y, class D> shared_ptr(unique_ptr<Y, D>&& r);
```

### 成员方法

*   `get()`: 返回管理对象的指针;
*   `swap()`: 交换管理对象;
*   `reset()`: 替换管理对象，原对象将销毁;
*   `use_count()`: 返回 shared_ptr 所指对象的引用计数;
*   `unique()`: 判断所管理对象是否仅由当前`shared_ptr`的实例管理;
*   `operator bool`: 检查是否有关联的管理对象;

下面为使用`shared_ptr`的示例:

```
#include <iostream>
#include <memory>
#include <string>

int main()
{
        std::string* str = new std::string("I'm shared_ptr.");
            
        // 1.创建shared_ptr
        std::shared_ptr<std::string> sptr1;
        std::shared_ptr<std::string> sptr2(str);
        sptr1 = sptr2;
        std::shared_ptr<std::string> sptr3(new std::string("Zhangsan"));
            
        // 2.使用右值unique_ptr创建shared_ptr
        std::shared_ptr<std::string> sptr4(std::unique_ptr<std::string>(
                        new std::string("I'm unique_ptr.")));
        std::cout << *sptr4 << std::endl;    

        // 3.get()返回管理对象的指针
        std::string* pstr = sptr1.get();
        std::cout << "*pstr: " << *pstr << std::endl;

        // 4.use_count()返回引用计数    
        std::cout << "sptr1.use_count: " << sptr1.use_count() << std::endl;
        std::cout << "sptr2.use_count: " << sptr2.use_count() << std::endl;
        std::cout << "sptr3.use_count: " << sptr3.use_count() << std::endl;

        // 5.unique()检查是否仅仅被该对象关联
        std::cout << "sptr1.unique: " << sptr1.unique() << std::endl;
        std::cout << "sptr3.unique: " << sptr3.unique() << std::endl;
            
        // 6.operator bool检查是否有关联的对象
        if(sptr1)
                std::cout << *sptr1 << std::endl;
            
        // 7.reset()替换管理对象
        sptr1.reset(new std::string("I'm new shared_ptr."));
        std::cout << *sptr1 << std::endl;    

        // 8.swap()交换管理对象
        sptr1.swap(sptr3);
        std::cout << *sptr1 << std::endl;
            
        // 9.使用std::make_shared(T&& t)创建shared_ptr.
        auto sptr5 = std::make_shared<std::string>("I'm from std::makeA_shared.");
        std::cout << *sptr5 << std::endl;

        return 0;
}

/*
@ubuntu:~$ g++ shrptr.cpp -o shrptr --std=c++11
@ubuntu:~$ ./shrptr 
I'm unique_ptr.
*pstr: I'm shared_ptr.
sptr1.use_count: 2
sptr2.use_count: 2
sptr3.use_count: 1
sptr1.unique: 0
sptr3.unique: 1
I'm shared_ptr.
I'm new shared_ptr.
Zhangsan
I'm from std::makeA_shared.
*/
```

weak_ptr
--------

`weak_ptr`是一种不控制所管理对象生存期的智能指针，也就是说，它对被管理对象具有弱引用性。它指向一个由`shared_ptr`管理的对象，并且将一个`weak_ptr`绑定到一个`shared_ptr`不会改变`shared_ptr`的引用计数，一旦最后指向被管理对象的`shared_ptr`被销毁，即使有`weak_ptr`指向对象，对象依旧被销毁释放。`weak_ptr`在访问所引用的对象前必须先转换为`std::shared_ptr`。

### 构造方法

`weak_ptr`构造函数如下:

```
constexpr weak_ptr() noexcept;
template<class Y> 
weak_ptr(shared_ptr<Y> const& r) noexcept;
weak_ptr(weak_ptr const& r) noexcept;
template<class Y> 
weak_ptr(weak_ptr<Y> const& r) noexcept;
weak_ptr(weak_ptr&& r) noexcept;    // C++14
template<class Y> 
weak_ptr(weak_ptr<Y>&& r) noexcept; // C++14
```

### 成员函数

*   `use_count()`: 返回`weak_ptr`所指向对象的引用计数;
*   `expired()`: 检查 ``weak_ptr`是否过期，若被管理对象已被删除则为true，否则为false,等价于`use_count() = 0`;
*   `lock()`: 创建新的`shared_ptr`对象，它共享被管理对象的所有权。若无被管理对象，即 *this 为空，则返回亦为空的`shared_ptr`.
*   `reset()`: 释放被管理对象的所有权;
*   `swap(weak_ptr& r)`: 交换`*this`与`r`的内容;

使用`weak_ptr`示例如下:

```
#include <iostream>
#include <memory>
#include <string>

class Person {
private:
        std::string name;
public: 
        Person(const std::string& str):name(str){}
        ~Person(){} 
        void show() const {
                std::cout << "Name: " << name << std::endl;
        }
};

int main()
{
        // 1.创建weak_ptr管理Person*
        std::weak_ptr<Person> wptr1;
        std::shared_ptr<Person> sptr1(new Person("Zhangsan"));
        wptr1 = sptr1;
        std::weak_ptr<Person> wptr2(wptr1);
        std::cout << sptr1.use_count() << std::endl;        

        // 2.访问所管理对象前，必须转换为shared_ptr.
        // wptr1->show();//Error
        std::shared_ptr<Person> sptr2(wptr1);
        sptr2->show();
        std::cout << "sptr2.use_count: " << sptr2.use_count() << std::endl;

        // 3.查看wptr1所管理对象的引用计数
        std::cout << "wptr1.use_count: " << wptr1.use_count() << std::endl;
     
        // 4.检查是否过期
        std::weak_ptr<Person> wptr3;
        {
                auto sptr2 = std::make_shared<Person>("Lisi");
                wptr3 = sptr2;
                std::cout << "wptr3.expired ? " << std::boolalpha << wptr3.expired();
                std::cout << ", use count: " << wptr3.use_count() << std::endl;
        }   
        std::cout << "wptr3.expired ? " << std::boolalpha << wptr3.expired() << std::endl;

        // 5.创建新的shared_ptr共享被管理对象
        std::cout << "wptr1.use count before lock(): " << wptr1.use_count() << std::endl;
        auto p = wptr1.lock();
        std::cout << "wptr2.use count after locked(): " << wptr1.use_count() << std::endl;
        p->show();

        // 6.交换被管理对象
        auto sptr3 = std::make_shared<Person>("XiaoQiang");
        std::weak_ptr<Person> wptr4(sptr3);
        wptr1.swap(wptr4);
        std::cout << "wptr4.use_count: " << wptr4.use_count() << std::endl;

        // 7.释放wptr1
        wptr1.reset();
        std::cout << "wptr1.expired ? " << std::boolalpha << wptr1.expired() << std::endl;

        return 0;
}
```

### `weak_ptr`的使用场景

`weak_ptr`最常见的用法，是和`shared_ptr`搭配使用，当`shared_ptr`用于类的成员时，如果存在循环引用，则将无法释放内存，请看下面示例:

```
#include <iostream>
#include <memory>

class B;
class A
{
private:
        std::shared_ptr<B> m_sptr;
public:
        void set_sptr(std::shared_ptr<B>& sptr) {
                m_sptr = sptr;
        }
        ~A() { std::cout << "A deleted." << std::endl;}
};

class B
{
private:
        std::shared_ptr<A> m_sptr;
public:
        void set_sptr(std::shared_ptr<A>& sptr) {
                m_sptr = sptr;
        }
        ~B() { std::cout << "B deleted." << std::endl;}
};

int main()
{   
        A* a = new A; 
        B* b = new B;
        std::shared_ptr<A> asptr(a);
        std::shared_ptr<B> bsptr(b);
        std::cout << "apstr.use_count: " << asptr.use_count() << std::endl;
        std::cout << "bsptr.use_count: " << bsptr.use_count() << std::endl;
            
        a->set_sptr(bsptr);
        b->set_sptr(asptr);

        std::cout << "asptr.use_count: " << asptr.use_count() << std::endl;
        std::cout << "bsptr.use_count: " << bsptr.use_count() << std::endl;
        
        return 0;
}    
/*
@ubuntu:~$ g++ shaptr.cpp -o shaptr --std=c++11
@ubuntu:~$ ./shaptr 
apstr.use_count: 1
bsptr.use_count: 1
asptr.use_count: 2
bsptr.use_count: 2
*/
```

在以上示例中，A,B 类分别带有`shared_ptr`的成员，并且通过`set_sptr(shared_ptr&)`互相共享管理对象，那么当程序执行完毕后，asptr 和 bsptr 将负责释放管理对象的内存，但此时由于引用计数为 2, 因此减 1, 从而导致`new`申请的 A 和 B 的内存没有释放，从运行结果来看，他们的析构函数未执行。

为避免这种问题，只需要将`shared_ptr`成员修改为`weak_ptr`即可，因为`weak_ptr`对管理对象具有弱引用性，并且将一个`weak_ptr`绑定到一个`shared_ptr`不会改变`shared_ptr`的引用计数。修改后运行结果如下：

```
@ubuntu:~$ ./shaptr 
apstr.use_count: 1
bsptr.use_count: 1
asptr.use_count: 1
bsptr.use_count: 1
B deleted.
A deleted.
```

总结
==

*   1. 当管理动态数组时，只能使用`unique_ptr`;
*   2. 当需要使用多个指向同一个对象的指针时，只能使用`shared_ptr`;
*   3. 当程序不需要多个指向同一个对象的指针时，建议使用`unique_ptr`;
*   4. 使用`new`或`new[]`分配内存时，才能使用`unique_ptr`, 使用`new[]`分配内存时，只能使用`unique_ptr`.