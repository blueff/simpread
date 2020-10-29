\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[www.jianshu.com\](https://www.jianshu.com/p/18307f614c5b)

[![](https://upload.jianshu.io/users/upload_avatars/4070621/9ea06d51-8cab-4b5a-86e2-cf4dc98f2586.png?imageMogr2/auto-orient/strip|imageView2/1/w/96/h/96/format/webp)](https://www.jianshu.com/u/081d2285374d)

0.1762020.02.14 17:35:51 字数 1,191 阅读 2,024

##### 一、定义与声明

###### 1\. 先定义结构体类型再单独进行变量定义

```
struct Student
{
    int Code;
    char Name\[20\];
    char Sex;
    int Age;
};
struct Student Stu;
struct Student StuArray\[10\];
struct Student \*pStru;
```

结构体类型是 struct Student，因此，struct 和 Student 都不能省略。但实际上，我用 codeblocks 运行时，下面变量的定义，不加 struct 也是可以的。

###### 2\. 紧跟在结构体类型说明之后进行定义

```
struct Student
{
    int Code;
    char Name\[20\];
    char Sex;
    int Age;
}Stu,StuArray\[10\],\*pStu;
```

这种情况时，后面还可以再定义结构体变量。

###### 3\. 在说明一个无名结构体变量的同时直接进行定义

```
struct
{
    int Code;
    char Name\[20\];
    char Sex;
    int Age;
}Stu,Stu\[10\],\*pStu;
```

这种情况下，之后不能再定义其他变量。

###### 4\. 使用 typedef 说明一个结构体变量之后再用新类名来定义变量

```
​
typedef struct
{
    int Code;
    char Name\[20\];
    char Sex;
    int Age;
}student;
Student Stu,Stu\[10\],\*pStu;
```

Student 是一个具体的结构体类型，唯一标识。这里不用再加 struct

###### 5\. 使用 new 动态创建结构体变量

使用 new 动态创建结构体变量时，必须是结构体指针类型。访问时，普通结构体变量使用使用成员变量访问符`"."`，指针类型的结构体变量使用的成员变量访问符为`"->"`。

*   注意：动态创建结构体变量使用后勿忘 delete。

```
#include <iostream>

using namespace std;

struct Student
{
    int Code;
    char Name\[20\];
    char Sex;
    int Age;
}Stu,StuArray\[10\],\*pStu;

int main(){

    Student \*s = new Student();  // 或者Student \*s = new Student;
    s->Code = 1;
    cout<<s->Code;

    delete s;
    return 0;
}
```

##### 二、结构体构造函数

###### 三种结构体初始化方法：

*   1\. 利用结构体自带的默认构造函数
*   2\. 利用带参数的构造函数
*   3\. 利用默认无参的构造函数

**要点：**什么都不写就是使用的结构体自带的默认构造函数，如果自己重写了带参数的构造函数，初始化结构体时如果不传入参数会出现错误。在建立结构体数组时, 如果只写了带参数的构造函数将会出现数组无法初始化的错误！！！下面是一个比较安全的带构造的结构体示例

```
struct node{
    int data;
    string str;
    char x;
    //注意构造函数最后这里没有分号哦！
  node() :x(), str(), data(){} //无参数的构造函数数组初始化时调用
  node(int a, string b, char c) :data(a), str(b), x(c){}//有参构造
};
```

```
//结构体数组声明和定义
struct node{
    int data;
    string str;
    char x;
    //注意构造函数最后这里没有分号哦！
  node() :x(), str(), data(){} //无参数的构造函数数组初始化时调用
  node(int a, string b, char c) :data(a), str(b), x(c){}//初始化列表进行有参构造
}N\[10\];
```

##### 三、结构体嵌套

正如一个类的对象可以嵌套在另一个类中一样，一个结构体的实例也可以嵌套在另一个结构体中。例如，来看以下声明：

```
struct Costs
{
    double wholesale;
    double retail;
};

struct Item
{
    string partNum;
    string description;
    Costs pricing;
}widget;
```

Costs 结构体有两个 double 类型成员，wholesale 和 retail。Item 结构体有 3 个成员，前 2 个是 partNum 和 description，它们都是 string 对象。第 3 个是 pricing，它是一个嵌套的 Costs 结构体。如果定义了一个名为 widget 的 Item 结构体，则图 3 说明了其成员。

![](http://upload-images.jianshu.io/upload_images/4070621-dece9e95d9fdc33d.png)

嵌套结构体访问的方式：

```
widget.partnum = "123A";
widget.description = "iron widget";
widget.pricing.wholesale = 100.0;
widget.pricing.retail = 150.0;
```

##### 四、结构体赋值与访问

*   **赋值**  
    初始化结构体变量成员的最简单的方法是使用初始化列表。初始化列表是用于初始化一组内存位置的值列表。列表中的项目用逗号分隔并用大括号括起来。

```
struct Date
{
    int day, month, year;
};
```

该声明定义 birthday 是一个 Date 结构体的变量，大括号内的值按顺序分配给其成员。所以 birthday 的数据成员已初始化，如图 2 所示。

![](http://upload-images.jianshu.io/upload_images/4070621-20ccb4470e86cf05.png)

也可以仅初始化结构体变量的部分成员。例如，如果仅知道要存储的生日是 8 月 23 日， 但不知道年份，则可以按以下方式定义和初始化变量：

这里只有 day 和 month 成员被初始化，year 成员未初始化。但是，如果某个结构成员未被初始化，则所有跟在它后面的成员都需要保留为未初始化。使用初始化列表时，C++ 不提供跳过成员的方法。以下语句试图跳过 month 成员的初始化。这是不合法的。

```
Date birthday = {23,8};
```

还有一点很重要，不能在结构体声明中初始化结构体成员，因为结构体声明只是创建一个新的数据类型，还不存在这种类型的变量。例如，以下声明是非法的：

```
Date birthday = {23,1983}; //非法
```

因为结构体声明只声明一个结构体 “看起来是什么样子的”，所以不会在内存中创建成员变量。只有通过定义该结构体类型的变量来实例化结构体，才有地方存储初始值。

*   **访问**

定义结构体：

```
//非法结构体声明
struct Date
{
    int day = 23,
    month = 8,
    year = 1983;
}；
```

一般结构体变量的访问方式：

```
struct MyTree{
    MyTree\*left;
    MyTree\*right;
    int val;
    MyTree(){}
    MyTree(int val):left(NULL),right(NULL),val(val){}
};
```

可见，结构体中的变量，可以直接通过`"."`操作符来访问。

而对于结构体指针而言：必须通过`"->"`符号来访问指针所指结构体的变量。

```
int main(){

    MyTree t;
    t.val = 1;
    cout<<t.val;

    return 0;
}
```

​

​

[![](https://upload.jianshu.io/users/upload_avatars/4070621/9ea06d51-8cab-4b5a-86e2-cf4dc98f2586.png?imageMogr2/auto-orient/strip|imageView2/1/w/100/h/100/format/webp)](https://www.jianshu.com/u/081d2285374d)

总资产 15 (约 1.09 元) 共写了 12.8W 字获得 171 个赞共 57 个粉丝