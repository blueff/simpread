\> 本文由 \[简悦 SimpRead\](http://ksria.com/simpread/) 转码， 原文地址 \[blog.csdn.net\](https://blog.csdn.net/u010164190/article/details/88315709)

  **shared\_ptr 很好地消除了显式的 delete 调用，如果读者掌握了它的用法，可以肯定 delete 将会在你的编程字典中彻底消失 。但这还不够，因为 shared\_ptr 的构造还需要 new 调用，这导致了代码中的某种不对称性。虽然 shared\_ptr 很好地包装了 new 表达式，但过多的显式 new 操作符也是个问题，用 make\_shared() 来消除显式的 new 调用。**

 **make\_shared() 函数可以接受最多 10 个参数，然后把它们传递给类型 T 的构造函数，创建一个 shared\_ptr<T> 的对 象并返回。make\_shared() 函数要比直接创建 shared\_ptr 对象的方式快且高效，因为它内部仅分配一次内存，消除了 shared\_ptr 构造时的开销。**

make\_shared() 栗子:

```
1.demo\_01.cpp
#include <iostream>
#include <vector>
using namespace std;
 
class StructA{
public:
  int i;
  string str;
  //StructA(): i(100/\*int()\*/){//初始化列表
  StructA(int i){
    this->i = i;
    cout<< "StructA(int), line = "<< \_\_LINE\_\_<< endl;
  };
 
  StructA(string str){
    this->str = str;
    cout<< "StructA(string), line = "<< \_\_LINE\_\_<< endl;
  };
  
  ~StructA(){ cout <<"~StructA "<< endl;};
  
  void show(){
    cout << "show() is Called. line = "<< \_\_LINE\_\_<< endl;
  }
} ;
 
int main(int argc, const char \* argv\[\]){
  //make\_shared自动申请对象及内存,自动释放,不用手动执行new和delete
  shared\_ptr<StructA> pA = make\_shared<StructA>(123);
  cout << "pA->i = " << pA->i << endl;
 
  shared\_ptr<StructA> pB = make\_shared<StructA>("Hello Kitty!");
  cout << "pB->str = " << pB->str << endl;
  pB->show();
 
  auto t = make\_shared<StructA>("Number One!");
  cout << "t->str = " << t->str << endl;
  t->show();
  
  return 0;
}
 
2.demo\_02.cpp
#include <iostream>
#include <vector>
#include <map> 
using namespace std;
 
class StructA{
public:
  int i;
  //StructA(): i(100/\*int()\*/){//初始化列表
  StructA(int i){
    this->i = i;
    cout<< "StructA(int), line = "<< \_\_LINE\_\_<< endl;
  };
  
  ~StructA(){ cout <<"~StructA "<< endl;};
  
  void show(){
    cout << "show() is Called. line = "<< \_\_LINE\_\_<< endl;
  }
  template<class T>
  T add(T x){
    return x;
  }
} ;
 
int main(){ 
  map<int, shared\_ptr<StructA>> mStrA;
  //插入map映射key-value数据
  mStrA\[0\] = make\_shared<StructA>(123);
  mStrA\[1\] = make\_shared<StructA>(456);
  mStrA.insert(pair<int, shared\_ptr<StructA>>(2,make\_shared<StructA>(789)));
  mStrA.insert(pair<int, shared\_ptr<StructA>>(3,make\_shared<StructA>(333)));
  
  mStrA\[0\]->show();
  //遍历方式1
  for(auto &mA : mStrA){
    cout << mA.first << " " << mA.second->i << endl;
    //mA.second->show();
  }
  cout << endl;
  
  //遍历方式2
  map<int, shared\_ptr<StructA>>::iterator iter;
  for(iter = mStrA.begin(); iter != mStrA.end(); iter++){
    cout << iter->first << " "<<iter->second->i<<endl;
    //iter->second->show();
  }
  return 0;
}
```