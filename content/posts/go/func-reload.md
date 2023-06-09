---
title: "Go 为什么不支持函数重载和参数默认值？"
date: 2021-12-31T12:55:16+08:00
toc: true
images:
tags: 
  - go
  - 为什么
---

大家好，我是煎鱼。

大家在初学习 Go 语言时，带着其他语言的习惯，总是会有些不习惯，感觉非常不能理解，直打问号。

其中一点就是 Go 语言不支持函数重载和参数默认值，觉得使用起来很不方便。

为此，在这篇文章中煎鱼就和大家一起来了解为什么，有又会怎么样。

## 函数重载

函数重载（function overloading），也叫方法重载。是某些编程语言（如 C++、C#、Java、Swift、Kotlin 等）具有的一项特性。

该特性**允许创建多个具有不同实现的同名函数**，对重载函数的调用会运行其适用于调用上下文的具体实现。

从功能上来讲，就是允许一个函数调用根据上下文执行不同的方法，达到调用同一个函数名，执行不同的方法。

一个简单的例子：

```c++
#include <iostream>

int Volume(int s) {  // 立方体的体积。
  return s * s * s;
}

double Volume(double r, int h) {  // 圆柱体的体积。
  return 3.1415926 * r * r * static_cast<double>(h);
}

long Volume(long l, int b, int h) {  // 长方体的体积。
  return l * b * h;
}

int main() {
  std::cout << Volume(10);
  std::cout << Volume(2.5, 8);
  std::cout << Volume(100l, 75, 15);
}
```

在上述例子中，实现了 3 个同名的 `Volume` 函数，但是 3 个函数的入参个数、类型均不一样，也代表了不同的实现目的。

在主函数 `main` 中，传入了不同的入参，编译器或运行时再进行内部处理，从程序上来看达到了调用不同函数的目的。

这就是函数重载，一函数多形态。

## 参数默认值

参数默认值，又叫缺省参数。指的是允许程序员设定缺省参数并指定默认值，**当调用该函数并未指定值时，该缺省参数将为缺省值来使用**。

一个简单的例子：

```c++
 int my_func(int a, int b, int c=12);
```

在上述例子中，函数 `my_func` 一共有 3 个变量，分别是：a、b、c。变量 c 设置了缺省值，也就是 12。

其调用方式可以为：

```c++
 // 第一种调用方式
 result = my_func(1, 2, 3);
 // 第二种调用方式
 result = my_func(1, 2);
```

在第一种方式中，就会正常的传入所有参数。在第二种方式，由于第三个参数 c 并没有传递，因此会直接使用缺省值 12。

这就是参数默认值，也叫缺省参数。

## 为什么不支持

### 美好

从上述的功能特性介绍来看，似乎非常的不错，能够节省很多功夫。像是 Go 语言的 context 库中的这些方法：

```golang
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

要是有函数重载，直接就 WithXXX 就好了，只需要关注传入的参数类型，也不用 “记” 那么多个方法名了。

有同学说，有参数默认值。那就可以直接设置在上面，作为 “最佳实践” 给到使用函数的人，岂不美哉。那怎么 Go 语言就不支持呢？

### 细思

其实这和设计理念，和对程序的理解有关系。说白了，就是你喜欢 “显式”，还是 “隐喻”。

函数重载和参数默认值，其实是不好的行为。调用者只看函数名字，可能没法知道，你这个默认值，又或是入参不同，会调用的东西，会产生怎么样的后果？

你可以观察一下自己的行为。大部分人都会潜意识的追进去看代码，看看会调到哪，缺省值的作用是什么，以确保可控。

### 敲定

这细思的可能，在 Go 语言中是不被允许的。Go 语言的**设计理念就是 “显式大于隐喻”，追求明确，显式**。

在 Go FAQ 《Why does Go not support overloading of methods and operators?》有相关的解释。

如下图：

![](https://files.mdnice.com/user/3610/582eac4e-ecd1-4fb5-bde7-b2cbcac7f809.png)

官方有明确提到两个观点：
- 函数重载：拥有各种同名但不同签名的方法有时是很有用的，但在实践中也可能是混乱和脆弱的。
- 参数默认值：操作符重载，似乎更像是一种便利，不是绝对的要求。没有它，程序会更简单。

这就是为什么 Go 语言不支持的原因。

## 总结

在这篇文章中，我们介绍了业内常见的编程语言的函数重载和参数默认值的概念和使用方法。也结合了 Go 语言自身的设计理念，说明了为什么不支持的原因。

你会希望 Go 语言支持这几个特性功能吗，欢迎在评论区留言讨论和交流：）

## 参考

- 维基百科（函数重载和缺省值定义）
- Frequently Asked Questions (FAQ)