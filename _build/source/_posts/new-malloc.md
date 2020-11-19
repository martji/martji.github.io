---
title: new/delete、malloc/free 杂谈
date: 2020-11-08 16:21:20
categories: 
- C/C++


---

很多从 Java 或者 Python 等语言转到 C/C++ 的时候，都会对内存管理，对象申请等问题非常疑惑，笔者今天尝试结合自己的切身经历进行一个简单的记录。

<!-- more -->

## 堆和栈

内存区域通常可以划分为以下几个类型：

- 栈：由编译器在需要的时候分配，通常包括：局部变量、参数等
- 堆：由程序手动分配、回收
- 全局/静态存储：存放全局变量和静态变量
- 常量存储：存放常量

一个简单的例子如下：

```
/* main.cpp */

int a = 0;  // 全局/静态存储
char *p1;  // 全局/静态存储

main() {
  int b;  // 栈
  char s[] = "abc";  // abc 在全局/静态存储
  char *p2;  // 栈
  char *p3 = "123456";  // 123456\0 在常量存储，p3 在栈

  static int c = 0； // 全局/静态存储

  p1 = (char *)malloc(10);  // 堆
}
```

回到堆和栈的概念本身，一个比较好的角度是：

- 栈对应的是自动变量（局部变量）：会在作用域（如函数作用域、块作用域等）结束后析构、释放内存。因为分配和释放的次序是刚好完全相反的，所以可用到堆栈先进后出（first-in-last-out, FILO）的特性，而 C++ 语言的实现一般也会使用到调用堆栈（call stack）来分配自动变量（但非标准的要求）；
- 堆对应的是自由存储：可以在函数结束后继续生存，所以也需要配合 delete 来手动析构、释放内存（也可使用智能指针避免手动 delete ）。由于分配和释放次序没有限制，不能使用堆栈这种数据结构做分配，实现上可能采用自由链表（free list）或其他动态内存分配机制；

更多关于堆和栈的更对信息，例如：管理方式、分配效率等，在此不做介绍。

## new/delete VS malloc/free

回到本文讨论的内容，首先需要明确的是：不管是 new 还是 malloc ，都是在堆上进行内存分配，所以都需要手动的进行内存的回收。

关于两者的区别，可以总结如下：

1. new/delete 是 C++ 的操作符，而 malloc/free 是 C 中的函数；
2. new 做两件事，一是分配内存，二是调用类的构造函数；同样，delete 会调用类的析构函数和释放内存。而 malloc 和 free 只是分配和释放内存；
3. new 建立的是一个对象，而 malloc 分配的是一块内存；new 建立的对象可以用成员函数访问，不要直接访问它的地址空间；malloc 分配的是一块内存区域，用指针访问，可以在里面移动指针；new 出来的指针是带有类型信息的，而 malloc 返回的是 void 指针；
4. new/delete 是保留字，不需要头文件支持；malloc/free 需要头文件库函数支持；

一个简单的例子如下：

```
class Obj {
public:
    Obj() { cout << "Initialization" << endl; }
    ~Obj() { cout << "Destroy" << endl; }
    void Initialize() { cout << "Initialization" << endl; }
    void Destroy() { cout << "Destroy" << endl; }
};

void UseMallocFree() {
    Obj *a = (Obj*)malloc(sizeof(obj));
    a->Intialize();
    // ...
    a->Destroy();
    free(a);
}

void UseNewDelete() {
    Obj *a = new Obj;
    //...
    delete a;
}
```

类 Obj 的函数 Initialize 模拟了构造函数的功能，函数 Destroy 模拟了析构函数的功能。函数 UseMallocFree 中，由于 malloc/free 不能执行构造函数与析构函数，必须调用成员函数 Initialize 和 Destroy 来完成初始化与清除工作。函数 UseNewDelete 则简单得多。

注意：new 和 delete 配套使用，new[] 和 delete[] 配套使用。