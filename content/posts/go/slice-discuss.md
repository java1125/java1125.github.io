---
title: "Go 切片这道题，吵了一个下午！"
date: 2021-12-31T12:55:06+08:00
toc: true
images:
tags: 
  - go
---

大家好，我是煎鱼。

前几天听到咱 Go 读者交流群里的小伙伴私聊我，表示他们在群里一直在讨论一个问题 slice 相关的问题，众说纷纭。

![来自煎鱼的聊天记录](https://files.mdnice.com/user/3610/f2cedcdf-8f6e-42f9-9009-6f1ee0f74f91.png)

今天和各位小伙伴们一起来研究一下，避免后续又踩一遍坑，共同进步！

## 问题代码

引起群内大范围讨论的代码如下：

```golang
func main() {
	sl := make([]int, 0, 10)
	var appenFunc = func(s []int) {
		s = append(s, 10, 20, 30)
		fmt.Println(s)
	}
	fmt.Println(sl)
	appenFunc(sl)
	fmt.Println(sl)
	fmt.Println(sl[:10])
}
```

你认为程序的输出结果是什么？

是如下的答案：

```
[]
[10 20 30]
[]
[]
```

对吗？

看上去很有道理，但错了。正确的结果是：

```
[]
[10 20 30]
[]
[10 20 30 0 0 0 0 0 0 0]
```

这下可把大家整懵了，为什么输出 `sl` 和 `sl[:10]` 的结果差别这么大，这与预期的输出结果不一致。

群内小伙伴的问题更明确了，疑惑点是：

```golang
	fmt.Println(sl)     
	fmt.Println(sl[:10]) 
```

上述代码中，**为什么第一个 `sl` 打印结果是空的，第二个 `sl` 给索引位置就能打印出来**？

也有小伙伴不断在尝试 `sl[:10]` 以外的输出，有没有因为一些边界值改变而导致不行。

例如：

```golang
fmt.Println(sl[:])
```

你认为这个对应的输出结果是什么？

是如下的答案：

```
[10 20 30 0 0 0 0 0 0 0]
```

对吗？

看上去很有道理，但错了。正确的结果是：

```
[]
```

是没有任何元素输出，这下大家更懵了。为什么 `sl[:]` 的输出结果为空？

再看看变量 `sl` 的长度和容量：

```
fmt.Println(len(sl), cap(sl))
```

输出结果：

```
0 10
```

长度竟然是 0 ...迷了？

## 挖掘原因

### 三个问题
在研究了问题代码的表象后，我们要进一步的挖掘问题的原因。

请思考如下三个问题：
1. 为什么打印 `sl[:10]` 时，结果包含了 10 个元素，还包含了函数闭包中插入的 10, 20, 30，之间有什么关系？
2. 为什么打印 `sl` 变量时，结果为空？
3. 为什么打印 `sl[:]` 时，结果为空。但打印 `sl[:10]` 就正常输出？

### 了解底层

要分析起源，我们就必须要再提到 slice（切片）的底层实现，slice 底层存储的数据结构指向了一个 array（数组）。

如下：

![slice 和 array 的友谊小船](https://files.mdnice.com/user/3610/6a025aba-38f0-4ac5-92bb-515d05adeb68.png)

对应的 Slice 在运行时的表现是 SliceHeader 结构体，定义如下：

```golang
type SliceHeader struct {
 Data uintptr
 Len  int
 Cap  int
}
```
- Data：指向具体的底层数组。
- Len：代表切片的长度。
- Cap：代表切片的容量。

核心要记住的是：slice 真正存储数据的地方，是一个数组。slice 的结构中**存储的是指向所引用的数组指针地址**。

### 分析原因

在了解 slice 的底层后，我们需要来分析问题的起源，也就是那段 Go 程序。

我们关注到 `appenFunc` 变量，他其实是一个函数，并且结果中我们所看到的 10, 20, 30，也只有这里有插入的动作。因此这是需要分析的。

如下：

```golang
func main() {
	sl := make([]int, 0, 10)
	var appenFunc = func(s []int) {
		s = append(s, 10, 20, 30)
	}
	appenFunc(sl)
	fmt.Println(sl)
	fmt.Println(sl[:10])
}
```

但为什么在 `appenFunc` 函数中所插入的 10, 20, 30 元素，就跑到外面的切片 `sl` 中去了呢？

这其实结合 slice 的底层设计和函数传递就明白了，**在 Go 语言中，只有值传递**：

![](https://files.mdnice.com/user/3610/3d2999da-e405-4a42-ba37-858487052c3b.png)

具体可详见我之前写的《[又吵起来了，Go 是传值还是传引用？](https://mp.weixin.qq.com/s/qsxvfiyZfRCtgTymO9LBZQ)》，有明确分析和说明。

实质上在调用 `appenFunc(sl)` 函数时，**实际上修改了底层所指向的数组**，自然也就会发生变化，也就不难理解为什么 10, 20, 30 元素会出现了。

那为什么 `sl` 变量的长度是 0，甚至有人猜测是不是扩容了，这其实和上面的问题还是一样，因为是值传递，自然也就不会发生变化。

要记住一个关键点：**如果传过去的值是指向内存空间的地址，是可以对这块内存空间做修改的**。反之，你也改不了。

至此，也就解决了我们的第一个大问题。

### 切片小优化

还剩下两个大问题，这似乎用上面的结论没法完整解释。虽说程序是诱因，但这块最直接的影响是和切片访问的小优化有关。

常用的访问切片我们会用：

```
s[low : high]
```

注意这里是：low、high。可没有用 len、cap 这种定性的词语，也就代表着这里取的值是可变的。

当是切片（slice）时，表达式 `s[low : high]` 中的 high，**最大的取值范围对应着切片的容量（cap），不是单纯的长度（len）**。因此调用 `fmt.Println(sl[:10])` 时可以输出容量范围内的值，不会出现越界。

相对的 `fmt.Println(sl)` 因为该切片 len 值为 0，没有指定最大索引值，high 则取 len 值，导致输出结果为空。

至此，第二和第三个大问题就解决了。

注：访问元素的定位在 Go 编译期就确定的了，相关逻辑可以在 compile 相关的代码中看到。

## 总结

在今天这篇文章中，我们结合了 Go 语言中切片的基本底层原理、值传递、边界值取值等进行了多轮探讨。

我们要牢记：**如果传过去的值是指向内存空间的地址，是可以对这块内存空间做修改的**。这在多种应用场景下都是适用的。

**所谓的最大取值范围**，除非官方给你写定 len 或 cap，否则不要过于主观的认为，因为他**会根据访问的数据类型和访问定位等改变**。

注：欢迎大家一起多多讨论，加我微信号：cJY0728，备注：加群。我拉你进读者交流群，和大家一起捣鼓技术，突破自己。

## 参考
- 来自读者 1v1 私聊
- 来自 Go 读者群