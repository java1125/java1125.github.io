---
title: "Go 数组比切片好在哪？"
date: 2021-09-17T12:43:08+08:00
images:
tags: 
  - go
---

大家好，我是煎鱼。

前段时间有播放一条快讯，就是 Go1.17 会正式支持切片（Slice）转换到数据（Array），不再需要用以前那种骚办法了，安全了许多。

但是也有同学提出了新的疑惑，在 Go 语言中，数组其实是用的相对较少的，甚至会有同学认为在 Go 里可以把数组给去掉。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9764cd832d1e4875953b66434beab8e4~tplv-k3u1fbpfcp-zoom-1.image)

数组相较切片到底有什么优势，我们又应该在什么场景下使用呢？

这是一个我们需要深究的问题，因此今天就跟大家一起来一探究竟，本文会先简单介绍数组和切片是什么，再进一步对数组的使用场景剖析。

一起愉快地开始吸鱼之路。

数组是什么
-----

Go 语言中有一种基本数据类型，叫数组。其格式为：`[n]T`。是一个包含 N 个类型 T 的值的数组。

基本声明格式为：

```
var a [10]int

```

代表的是声明了一个变量 a 是一个包含 10 个整数的数组。数组的长度是其类型的一部分，所以数组不能被随意调整大小。

在使用例子上：

```
func main() {
 var a [2]string
 a[0] = "脑子进"
 a[1] = "煎鱼了"
 fmt.Println(a[0], a[1])
 fmt.Println(a)

 primes := [6]int{2, 3, 5, 7, 11, 13}
 fmt.Println(primes)
}

```

输出结果：

```
脑子进 煎鱼了
[脑子进 煎鱼了]
[2 3 5 7 11 13]

```

在赋值和访问上，数组可以针对不同的索引，进行单独操作。在内存布局上，数组的索引 0 和 1...是会在相邻区域，可直接访问。

切片是什么
-----

为什么数组在业务代码似乎用的很少。因为 Go 语言有一个切片的数据类型：

基本声明格式为：

```
var a []T

```

代表的是变量 a 是带有类型元素的切片T。通过指定两个索引（下限和上限）并用冒号隔开来形成切片：

```
a[low : high]

```

在使用例子上：

```
func main() {
 primes := [3]string{"煎鱼", "搞", "Go"}

 var s []string = primes[1:3]
 fmt.Println(s)
}

```

输出结果：

```
[搞 Go]

```

切片支持动态的扩缩容，不需要用户侧去关注，非常便利。更重要的一点是，切片的底层数据结构中本身就包含了数组：

```
type slice struct {
 array unsafe.Pointer
 len   int
 cap   int
}

```

也就很多人笑称：**在 Go 语言中数组已经可以下岗了，用切片就完事了**...

你怎么看待这个说法的呢，快速思考你心中的答案。

数组的优势
-----

在风尘仆仆介绍完数组和切片的基本场景后，在数组的优势方面，先了解一下官方的自述：

> > Arrays are useful when planning the detailed layout of memory and sometimes can help avoid allocation, but primarily they are a building block for slices.

非常粗暴间接：在规划内存的详细布局时，数组是很有用的，有时可以帮助避免分配，但主要是它们是分片的构建块。

我们再进一步解读，看看官方这股 “密文” 具体指的是什么，我们将该密文解读为以下内容进行讲解：

*   可比较。
    
*   编译安全。
    
*   长度是类型。
    
*   规划内存布局。
    
*   访问速度。
    

### 可比较

数组是固定长度的，它们之间是可以进行比较的，数组是值对象（不是引用或指针类型），你不会遇到 interface 等比较的误判：

```
func main() {
 a1 := [3]string{"脑子", "进", "煎鱼了"}
 a2 := [3]string{"煎鱼", "进", "脑子了"}
 a3 := [3]string{"脑子", "进", "煎鱼了"}

 fmt.Println(a1 == a2, a1 == a3)
}

```

输出结果：

```
false true

```

另一方面，切片不可以直接比较，也不能用于判断：

```
func main() {
 a1 := []string{"脑子", "进", "煎鱼了"}
 a2 := []string{"煎鱼", "进", "脑子了"}
 a3 := []string{"脑子", "进", "煎鱼了"}

 fmt.Println(a1 == a2, a1 == a3)
}

```

输出结果：

```
# command-line-arguments
./main.go:10:17: invalid operation: a1 == a2 (slice can only be compared to nil)
./main.go:10:27: invalid operation: a1 == a3 (slice can only be compared to nil)

```

同时数组可以作为 map 的 k（键），而切片不行，切片并没有实现平等运算符（equality operator），需要考虑的问题有非常多，例如：

*   涉及浅层与深层比较。
    
*   指针与值比较。
    
*   如何处理递归类型。
    

平等是为结构体和数组定义的，所以这类类型可以作为 map 键使用。切片没有平等的定义，有着非常根本的差距。

数组的可比较和平等，切片做不到。

### 编译安全

数组可以提供更高的编译时安全，可以在编译时检查索引范围。如下：

```
s := make([]int, 3)
s[3] = 3 // "Only" a runtime panic: runtime error: index out of range

a := [3]int{}
a[3] = 3 // Compile-time error: invalid array index 3 (out of bounds for 3-element array)

```

这个编译检查的帮助虽 “小”，但其实非常有意义。我是日常看到各大切片越界的告警，感觉都能背下来了...

万一这个越界是在 hot path 上，影响大量用户，分分钟背个事故，再来个 3.25，岂不梦中惊醒？

数组的编译安全，切片做不到。

### 长度是类型

数组的长度是数组类型声明的一部分，因此**长度不同的数组是不同的类型**，两个就不是一个 “东西”。

当然，这是一把双刃剑。其优势在于：可用于显式指定所需数组的长度。

例如：你在业务代码中想编写一个使用 IPv4 地址的函数。可以声明 `type [4]byte`。使用数组有以下意识：

*   有了编译时的保证，也就是达到传递给你的函数的值将恰好具有4个字节，不多也不少的效果。
    
*   如果长度不对，也就可以认为是无效的 IPv4 地址，非常方便。
    

同时数组的长度，也可以用做记录目的：

*   MD5 类型，在 `crypto/md5`包中，`md5.Sum` 方法返回类型为的值，`[Size]byte` 其中 `md5.Size` 一个常量为16：MD5 校验和的长度。
    
*   IPv4 类型，所声明的 `[4]byte` 正确记录了有 4 个字节。
    
*   RGB 类型，所声明的 `[3]byte` 告诉有对每个颜色成分 1 个字节。
    

在特定业务场景上，使用数组更好。

### 规划内存布局

数组可以更好地控制内存布局，因为不能直接在带有切片的结构中分配空间，所以可以使用数组来解决。

例如：

```
type Foo struct {
    buf [64]byte
}

```

不知道你是否有在一些 Go 图形库上见过这种不明所以的操作，例子如下：

```
type TGIHeader struct {
    _        uint16 // Reserved
    _        uint16 // Reserved
    Width    uint32
    Height   uint32
    _        [15]uint32 // 15 "don't care" dwords
    SaveTime int64
}

```

因为业务需求，我们需要实现一个格式，其中格式是 "TGI"（理论上的Go Image），头包含这样的字段：

*   有 2 个保留字（每个16位）。
    
*   有 1 个字的图像宽度。
    
*   有 1 个字的图像高度。
    
*   有 15 个业务 "不在乎 "的字节。
    
*   有 1 个保存时间，图像的保存时间为8字节，是自1970年1月1日UTC以来的纳秒数。
    

这么一看，也就不难理解数组的在这个场景下的优势了。定长，可控的内存，在计划内存布局时非常有用。

### 访问速度

使用数组时，其访问（单个）数组元素比访问切片元素更高效，时间复杂度是 O（1）。例如：

```
 var a [2]string
 a[0] = "脑子进"
 a[1] = "煎鱼了"
 fmt.Println(a[0], a[1])

```

切片就没那么方便了，访问某个位置上的索引值，需要：

```
 var a []int{0, 1, 2, 3, 4, 5}  
  number := numbers[1:3]

```

相对复杂些的，删除指定索引位上的值，可能还有小伙伴纠结半天，甚至在找第三方开源库想快速实现。

无论在访问速度和开发效率上，数组都占一定的优势，这是切片所无法直接对比的。

总结
--

经过一轮的探讨，我们对 Go 语言的数组有了更深入的理解。总结如下：

*   数组是值对象，可以进行比较，可以将数组用作 map 的映射键。而这些，切片都不可以，不能比较，无法作为 map 的映射键。
    
*   数组有编译安全的检查，可以在早起就避免越界行为。切片是在运行时会出现越界的 panic，阶段不同。
    
*   数组可以更好地控制内存布局，若拿切片替换，会发现不能直接在带有切片的结构中分配空间，数组可以。
    
*   数组在访问单个元素时，性能比切片好。
    
*   数组的长度，是类型的一部分。在特定场景下具有一定的意义。
    
*   数组是切片的基础，每个数组都可以是一个切片，但并非每个切片都可以是一个数组。如果值是固定大小，可以通过使用数组来获得较小的性能提升（至少节省 slice 头占用的空间）。
    

与你心目中的数组的优势是否一致呢，欢迎大家在评论区进行讨论和交流。

我是煎鱼，咱们下期再见：）


参考
--

*   In GO programming language what are the benefits of using Arrays over Slices?
    
*   Why have arrays in Go?