---
title: "Effective C++ 学习笔记"
date: 2019-09-27T21:39:39+08:00
tags: [c++]
---

本文参考书目：Effective C++ by Scott Meyers, Effective Modern C++ by Scott Meyers, Linux 多线程服务端编程 by 陈硕。

本文记录一些以前未思考的准则。

### 构造函数不注册回调函数

形如

```c++
class Foo: public Observer {
 public:
  Foo (Observable *s)
  { s->register(this); }
  virtual void update();
}
```

线程安全的问题如下（参考[知乎](https://www.zhihu.com/question/23984289)）

- 未构造完成即调用。
- 回调函数使用 this 指针，如果正在执行基类的构造函数，而派生类仍未构造，虚函数就会有问题。

推荐方法：二段式构造 constructor + initialize()

### 何时声明虚析构函数

析构函数声明为虚函数的好处很显然，但滥用虚析构函数也会带来问题。准则是：只有当类中含有至少一个虚函数，才声明析构虚函数。

问题：由于使用虚函数，会引入虚函数表改变最初的数据结构，使得这个类失去可移植性。

