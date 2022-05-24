---
layout:     post
title:      "C++虚函数 "
subtitle:   ""
date:       2022-05-23 22:20:00
author:     "d"
# header-img: "img/post-bg-android.jpg"
catalog: true
tags: 
    - C++
    - 虚函数
---

### Intuition

提到C++和多态，就难免不提及虚函数这一内容，虽然我觉得对于普通C++程序员，应该把更多精力放在业务而不是某个语言本身，但是虚函数在C++编程上确实是一个经常使用以至于不能避而不谈的特性或功能，以至于在面试过程中面试官常常把这一点作为应聘者是否基本掌握C++语言的知识点之一，因此有必要写篇文章进行整理。

### C++ 中的多态

多态是面向对象语言的特性之一，是指一个接口可以表现出来多个功能，举例而言，一个人在社会上会有多种角色：父亲、职工、公民...对于C++来说，多态有两个时期：

**运行时期：** 在运行时才确定接口要执行哪个功能，在C++中依赖虚函数实现。

**编译时期：** 在编译时期就能确定接口中哪个方法要执行。

说完多态产生的时期，再说多态的类型。
**Ad-hoc：**（通常把它翻译成[特设多态](https://zh.m.wikipedia.org/zh-hans/%E7%89%B9%E8%AE%BE%E5%A4%9A%E6%80%81)，但是又不够直观，所以一般就直接用英文），具体方式有重载(overloading)和模版特例化(template specialisation)，

对于*函数重载*：

```C++
void fun(int m) {
  
}
void fun(double m) {

}
// main
double m = 6.5;
fun(m);
```
会调用fun(double)，通过汇编可以看到，

```C++
0000000100003f50 <__Z3funi>:
...

0000000100003f60 <__Z3fund>:
...

0000000100003f70 <_main>:
...
100003f9c: f1 ff ff 97 	bl	0x100003f60 <__Z3fund>
...
```
fun()接口有两种形式__Z3funi和__Z3fund，通过参数类型区分，值得一提的是，Z3表示函数名长度，在主函数中，调用了__Z3fund，即fun(double)。C语言是不能函数重载的，因为C编译器不能通过参数区分不同的接口类型。

*模版特例化* 也很容易理解
```C++
templete<typename T>
void fun(T a) { std::cout<< "模板" << std::endl; }

templete<>
void fun(int a) { std::cout<< "特例化int模板" << std::endl; }

// main
fun<char>fun('a'); // "模板" 
fun<int>fun(0);    // "特例化int模板"
```

**参数多态：** 模版传入不同类型的参数，就能执行不同的功能，比如下例，传入int类型，他就是功能为执行两数之和的函数，传入string，它就是功能为字符串拼接的函数，即 *“如果一个鸟叫声像鸭子，走路像鸭子，游泳像鸭子，那么它就是鸭子”* 。
```C++
template<typename T>
void fun(T a, T b) {
  std::cout << a + b << std::endl;
}

// main
int a = 1, b = 2;
fun<int>(a, b); // 3
string m = "ab", n = "cd";
fun<string>(m, n); // "abcd"
```

**动态多态：** 通过虚函数实现，在运行时期决定执行同一接口的哪个功能。
首先看包含虚函数的类的内存布局:
```C++
class A {
public:
  virtual void fun() {
    std::cout << "A fun" << std::endl;
  }
  virtual void fun2() {
    std::cout << "A fun2" << std::endl;
  }
private:
  int m_;
};
```
内存布局如下：
```C++
0 | class A
0 |   (A vtable pointer)
8 |   int m_
| [sizeof=16, dsize=12, align=8,
|  nvsize=12, nvalign=8]
```
可以看到，在起始地址是一个8个字节(64位机器)的指针vtable pointer，接着是成员变量m_，这个虚拟指针指向虚函数表，虚函数表类似一个数组，存放了虚函数的地址，内存布局如下：
![类A虚函数内存布局](/img/virtual_fun_1.png)
下面进行验证：
```C++
typedef void(*vir_fun_ptr)(void);
int main() {
    A a;
    // 因为虚指针在对象起始处，所以对其取指针，获得虚函数指针
    uint64_t* vir_ptr_a = reinterpret_cast<uint64_t*>(&a); 
    // 获取虚指针中存储的内容，即虚函数表的地址
    uint64_t* vir_table_addr_a = reinterpret_cast<uint64_t*>(*vir_ptr_a);

    vir_fun_ptr vf_a;

    // 虚函数表地址起始处存储第一个虚函数的地址
    vf_a = reinterpret_cast<vir_fun_ptr>(*vir_table_addr_a);
    vf_a(); // 输出 "A fun"
    // 虚函数表地址偏移一位则是第二个虚函数的地址
    vf_a = reinterpret_cast<vir_fun_ptr>(*(vir_table_addr_a + 1));
    vf_a(); // 输出 "A fun2"
    
    return 0;
}
```
这样，就验证了对象中虚指针的布局和虚函数表中虚函数的布局。

下面，再看一下存在继承关系时，虚函数是如何动态绑定的，类B继承了类A，其内存布局如下：
![类B虚函数内存布局](/img/virtual_fun_2.png)
类B地址起始处也是虚指针，指向类B的虚函数表，它继承自类A，类B虚函数表的起始地址也存储了类A的fun函数地址，由于类B 重写了类A的fun2()，所以其虚函数表地址偏移1位的地址存储了类B自己的fun2()的函数地址。验证如下：
```C++
typedef void(*vir_fun_ptr)(void);
int main() {
    A a;
    A* b = new B();

    // 对象a和对象b的虚指针
    uint64_t* vir_ptr_a = reinterpret_cast<uint64_t*>(&a); 
    uint64_t* vir_ptr_b = reinterpret_cast<uint64_t*>(b); 

    // 对象a和对象b的虚函数表的地址
    uint64_t* vir_table_addr_a = reinterpret_cast<uint64_t*>(*vir_ptr_a);
    uint64_t* vir_table_addr_b = reinterpret_cast<uint64_t*>(*vir_ptr_b);

    vir_fun_ptr vf_a;
    vir_fun_ptr vf_b;


    // 对象a和对象b虚函数表存储的第一个虚函数的地址
    // 证明了两者存储的第一个虚函数是同一个(函数地址相同)
    vf_a = reinterpret_cast<vir_fun_ptr>(*vir_table_addr_a);
    vf_a(); // 输出 "A fun"
    vf_b = reinterpret_cast<vir_fun_ptr>(*vir_table_addr_b);
    vf_b(); // 输出 "A fun"

    // 对象a和对象b虚函数表存储的第二个虚函数的地址
    // 当子类重写父类虚函数时，子类会存放自己的虚函数（函数地址）
    vf_a = reinterpret_cast<vir_fun_ptr>(*(vir_table_addr_a + 1));
    vf_a(); // 输出 "A fun2"
    vf_b = reinterpret_cast<vir_fun_ptr>(*(vir_table_addr_b + 1));
    vf_b(); // 输出 "B fun2"
    
    return 0;
}
```
综上，每个有虚函数的类会有一个虚函数表，存放了虚函数的地址，它的对象共享这个虚函数表，对象内存地址起始处是一个虚函数指针，指向这个虚函数表。当发生继承时，子类会继承父类的虚函数表，如果重写了父类的虚函数，则会用自己的重写虚函数地址在虚函数表对应的位置替代。

### Reference
[1] [Polymorphism in C++](https://www.geeksforgeeks.org/polymorphism-in-c/)
