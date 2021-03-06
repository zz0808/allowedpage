---
layout:     post
title:      "C++中的static关键字"
subtitle:   ""
date:       2022-05-19 12:00:00
author:     "d"
# header-img: "img/post-bg-android.jpg"
# catalog: true
tags: 
    - C++
---

在c++中static关键字可以用来修饰变量和函数，下面分别介绍。

#### 1.修饰变量

如果显式初始化，存储在data段，否则存储在.bss段。

##### 1.1修饰一个编译单元中变量

这个变量存在于它所在的编译单元中，并且和该编译单元生命周期同步，只在编译单元内可见。

所谓**编译单元**，即.cpp/.cc文件，编译器只会编译.cpp/.cc等文件，不编译.h/.hpp。在编译的时候，会将编译单元编译成目标文件(.o)再进行链接。

下面聊一下为什么static修饰的变量只在编译单元可见。不同的目标文件链接在一起的时候，会涉及到**符号**的重定位（因为多个目标文件链接在一起时各个目标文件中符号的地址会发生变化），每个变量和函数都有自己的名称，通常称为符号，当我们想获取变量值时，就从变量符号读取内容，想调用函数时，就进行函数符号调用。各个目标文件通过链接进行整合，最终生成共享库或可执行文件（下篇文章我会写一下c++编译/链接的详细过程），**对于静态变量符号**，会禁止它被外部链接从而只能在本编译单元可见，因此链接器不会从全局的角度重定位它的地址，而是基于本编译单元的数据段(静态变量存储在数据段)的地址进行重定位。

##### 1.2修饰函数中的局部变量

它的生命周期和本编译单元是同步的，其他和普通局部变量相同，它们只会在执行到对应语句时初始化一次，根据这个特性，可以用于实现单例模式：

```
class Singleton {
public:
    Singleton(const Singleton&)=delete;
    Singleton& operator=(const Singleton&)=delete;
    Singleton& GetInstance() {
        // c++11规定了局部对象在初始化完成前不允许其他线程访问，因此可保证线程安全
        static Singleton instance;
        return instance;
    }
private:
    Singleton()=default;
    ~Singleton()=default;
};
```

##### 1.3修饰类成员

它是属于类而不是对象的，被实例后的对象共享，因此也有人把它看作对象间交流的一种方式，通过

```
作用域::类名::成员
```

的方式访问。一般我们会在类中进行声明，在类外（编译单元中）定义。

#### 2.函数

##### 2.1类中的静态函数

作为类中的静态函数，可以用来访问静态成员变量，不能访问非静态成员，因为他们不属于任何一个对象，所以调用静态成员函数时没法把当前的对象作为参数传进去，另外说一句，当调用普通成员函数时，比如：

```
class A {
public:
// ...
    void fun(int a, int b);
};
// 调用
A a;
a.fun(m, n);
// 实际上是执行
fun(m, n, this);
// 这个this即当前对象a，也就是把当前对象的指针传进去了，所以对象调用普通成员函数
//可以访问不同对象的成员变量
```

##### 2.2 普通静态成员函数

它们不能被其他编译单元使用，因为链接过程会把它们忽略掉，当你确保某个函数只会在本编译单元使用时，可以用static修饰它，这样能减少链接时链接器的工作量。

#### 总结

static关键字修饰的变量或函数只在本编译单元可见，是因为链接器在链接时会忽略它们的链接，生命周期伴随编译单元，static变量只会被初始化一次，存储在data段或bss段，static成员函数属于类而非某个对象，因此不能访问非静态成员变量，合理使用全局静态函数能减少链接器的工作量。

以上。

第一次写文章，难免表述有不清晰/不正确的地方，希望大家多多交流指正。
