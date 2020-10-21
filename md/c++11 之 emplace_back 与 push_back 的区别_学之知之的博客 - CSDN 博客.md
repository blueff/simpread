\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[blog.csdn.net\](https://blog.csdn.net/p942005405/article/details/84764104)

c++ 开发中我们会经常用到插入操作对 stl 的各种容器进行操作，比如 vector,map,set 等。在引入右值引用，转移构造函数，转移复制运算符之前，通常使用 push\_back()向容器中加入一个右值元素 (临时对象) 时，首先会调用构造函数构造这个临时对象，然后需要调用拷贝构造函数将这个临时对象放入容器中。原来的临时变量释放。这样造成的问题就是临时变量申请资源的浪费。   
引入了右值引用，转移构造函数后，push\_back() 右值时就会调用构造函数和转移构造函数, 如果可以在插入的时候直接构造，就只需要构造一次即可。这就是 c++11 新加的 emplace\_back。

emplace\_back 函数原型：

```
template <class... Args>
  void emplace\_back (Args&&... args);
```

在容器尾部添加一个元素，这个元素原地构造，不需要触发拷贝构造和转移构造。而且调用形式更加简洁，直接根据参数初始化临时对象的成员。  
一个很有用的例子:

```
#include <vector>  
#include <string>  
#include <iostream>  
 
struct President  
{  
    std::string name;  
    std::string country;  
    int year;  
 
    President(std::string p\_name, std::string p\_country, int p\_year)  
        : name(std::move(p\_name)), country(std::move(p\_country)), year(p\_year)  
    {  
        std::cout << "I am being constructed.\\n";  
    }
    President(const President& other)
        : name(std::move(other.name)), country(std::move(other.country)), year(other.year)
    {
        std::cout << "I am being copy constructed.\\n";
    }
    President(President&& other)  
        : name(std::move(other.name)), country(std::move(other.country)), year(other.year)  
    {  
        std::cout << "I am being moved.\\n";  
    }  
    President& operator=(const President& other);  
};  
 
int main()  
{  
    std::vector<President> elections;  
    std::cout << "emplace\_back:\\n";  
    elections.emplace\_back("Nelson Mandela", "South Africa", 1994); //没有类的创建  
 
    std::vector<President> reElections;  
    std::cout << "\\npush\_back:\\n";  
    reElections.push\_back(President("Franklin Delano Roosevelt", "the USA", 1936));  
 
    std::cout << "\\nContents:\\n";  
    for (President const& president: elections) {  
       std::cout << president.name << " was elected president of "  
            << president.country << " in " << president.year << ".\\n";  
    }  
    for (President const& president: reElections) {  
        std::cout << president.name << " was re-elected president of "  
            << president.country << " in " << president.year << ".\\n";  
    }
 
}
```

输出

```
emplace\_back:
I am being constructed.
 
push\_back:
I am being constructed.
I am being moved.
 
Contents:
Nelson Mandela was elected president of South Africa in 1994.
```

网上有人说尽量使用 emplace\_back 代替 push\_back 有没有什么特例是不能替换的呢，搜了一下发现了一个例子：

[emplace\_back 造成的引用失效](https://blog.csdn.net/wangshubo1989/article/details/50358044) 

**_勘误：window visual studio 2015 编译下面程序会出现 引用失效问题，而 linux gcc 和 qt 等编译环境中未出现下面问题。感谢大家的指正。_**
===========================================================================================

```
#include <vector>
#include <string>
#include <iostream>
using namespace std;
 
int main()
{
    vector<int> ivec;
    ivec.emplace\_back(1);
    ivec.emplace\_back(ivec.back());
    for (auto it = ivec.begin(); it != ivec.end(); ++it)
        cout << \*it << " ";
    return 0;
}
 
//输出：
1 -572662307
```

尝试 1：不直接给 emplace\_back 传递 ivec.back()：

```
#include <vector>
#include <string>
#include <iostream>
using namespace std;
 
int main()
{
    vector<int> ivec;
    ivec.emplace\_back(1);
    auto &it = ivec.back();
    ivec.emplace\_back(it);
    for (auto it = ivec.begin(); it != ivec.end(); ++it)
        cout << \*it << " ";
    return 0;
}
输出：
1 -572662307
```

尝试 2：不给 emplace\_back 传递引用：

```
#include <vector>
#include <string>
#include <iostream>
using namespace std;
 
int main()
{
    vector<int> ivec;
    ivec.emplace\_back(1);
    auto it = ivec.back();
    ivec.emplace\_back(it);
    for (auto it = ivec.begin(); it != ivec.end(); ++it)
        cout << \*it << " ";
    return 0;
}
输出：
1 1
```

我们如愿以偿，这时候应该可以得到结论了，ivec.back() 返回的是引用，但是这个引用失效了，所以才会输出不正确；我们之前也提到过，重新分配内存会造成迭代器的失效，这里是造成了引用的失效。

再回头看看 emplace\_back 的描述：   
if a reallocation happens, all iterators, pointers and references related to this container are invalidated.   
Otherwise, only the end iterator is invalidated, and all other iterators, pointers and references to elements are guaranteed to keep referring to the same elements they were referring to before the call.

进一步  
尝试 3：避免 emplace\_back 引起重新分配内存：

```
#include <vector>
#include <string>
#include <iostream>
using namespace std;
 
int main()
{
    vector<int> ivec;
    ivec.reserve(4);
    ivec.emplace\_back(1);
    ivec.emplace\_back(ivec.back());
    for (auto it = ivec.begin(); it != ivec.end(); ++it)
        cout << \*it << " ";
    return 0;
}
输出：
1 1
```

###   
参考链接：

https://blog.csdn.net/windpenguin/article/details/75581552 

https://blog.csdn.net/xiaolewennofollow/article/details/52559364 

https://blog.csdn.net/wangshubo1989/article/details/50358044