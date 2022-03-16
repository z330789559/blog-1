---
title: "Golang Generic Programming"
date: 2022-03-16T22:30:06+08:00
---

![](https://tva1.sinaimg.cn/large/e6c9d24egy1h0c3879fsyj21hc0u077g.jpg)

### Go1.8 泛型

最近随着`Go 1.18`版本发布了，也就是`Go`正式支持`generic`编程了，这个版本的发布将会给你`Go`编程开发体验带来巨大变动，估计后面一些开源库也会陆续重构，对`Go`语言从发布至今应该是一次最大改动。

![官方推特消息](https://tva1.sinaimg.cn/large/e6c9d24egy1h0brghg5tuj20u00x077u.jpg)


### Generic Programming

在之前的版本中`Go`语言想做一些通用的数据类型的编程操作的时候，可能大部分还是使用`interface`来进行编程，但是代码里面会出现各种断言操作，并且还有预判出可能需要的数据类型，重复编写一些只是`类型`不一样的代码块，如下图就是之前在社区里面经常流行一张关于`Go`泛型编程调侃图👇🏻，只是数据参数类型不一样，但是具体逻辑代码又是一样的，会使得开发者我们重复编写一些代码块：

![在没有泛型编程之前的Go](https://tva1.sinaimg.cn/large/e6c9d24egy1h0brydd6vcg20f40gcdvr.gif)

当然之前版本`Go`语言标准库中提供一个`text/template`包，这个包如果开发者自己能力很强可以通过自己写一些代码块，然后通过模板引擎去生成相关的代码，一些第三方的框架代码生成就是这个原理，例如官方的`go generate`命令。

其实泛型编程，可以简单理解毕昇发明的活字印刷术一样，我在编写程序的时候把一些通用的代码逻辑写好，只是可能运行的时候数据类型可能不同而已，只需要开发者定义好预计数据类型，然后其他事情交给`Go`编译器。

![类似于原理](https://tva1.sinaimg.cn/large/e6c9d24egy1h0bsmrbukyg20b706bqv5.gif)

我本人也是对`Go`泛型编程也是很期待，在`2020`的时候我在实现一些通用的算法和数据结构时就遇到了一些重复代码逻辑的问题，并且向官方询问过，当时官方给了我一些泛型的设计草案，后面这个草案一直在改动，直到现在终于发布了。

![issue](https://tva1.sinaimg.cn/large/e6c9d24egy1h0bssx0finj211l0u0q6z.jpg)

- [https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md)

### 体验一把泛型

首先需要使用泛型的话，你的`Go`语言版本需要升级到`1.8`这个语义化版本，如下图我已经升级到这个版本了。

![升级到Go1.8](https://tva1.sinaimg.cn/large/e6c9d24egy1h0bt5i9kanj21mc0ikwhg.jpg)

下面我写一段冒泡排序的代码，但是这个`bubbleSort`函数参数类型只能为`int64`，也就是说我们只能传入一个类型为`int64`切片，如果我需要传入其他类型的数据，那么我们就需要重新写一个函数逻辑是相同的代码片段但是类型又不同，这就是没有泛型带来的痛苦。

```go
package main

import "fmt"

func bubbleSort(sequence []int64) {
    for i := 0; i < len(sequence)-1; i++ {
        for j := 0; j < len(sequence)-1-i; j++ {
            if sequence[j] > sequence[j+1] {
                sequence[j], sequence[j+1] = sequence[j+1], sequence[j]
            }
        }
    }
}

func main() {
    var sequence = []int64{12, 11, 100, 99, 88, 23}
    bubbleSort(sequence)
    fmt.Println(sequence)
}
```
有时候在开发的时候，为了统一使用一个函数处理，没有泛型情况下，需要对类型进行断言处理，例如在某种情况下居然写出下面这样的代码：

```go
func bubbleSortByInterface(sequence []interface{}) {
    switch sequence[0].(type) {

    case int64:

        var arr []int64
        for _, val := range sequence {
            arr = append(arr, val.(int64))
        }

        for i := 0; i < len(arr)-1; i++ {
          for j := 0; j < len(arr)-1-i; j++ {
                if arr[j] > arr[j+1] {
                    arr[j], arr[j+1] = arr[j+1], arr[j]
                }
          }
        }
        fmt.Println(arr)
    case float64:

        var arr []float64
        for _, val := range sequence {
            arr = append(arr, val.(float64))
        }

        for i := 0; i < len(arr)-1; i++ {
          for j := 0; j < len(arr)-1-i; j++ {
              if arr[j] > arr[j+1] {
                arr[j], arr[j+1] = arr[j+1], arr[j]
              }
          }
        }
        fmt.Println(arr)
    default:
        panic("type not support!!!")
    }
}
```

然后就可以使用`interface`一把梭了：

```go
func main() {
    var sequence = []interface{}{12.12, 1.1, 99.4, 99.2, 88.8, 2.3}
    bubbleSortByInterface(sequence)
    // [1.1 2.3 12.12 88.8 99.2 99.4]
}
```
但是这种一不注意就出现各种`bug`并且代码量上去就很难阅读代码。

### 泛型版本

接下来就是`Go Generic`的使用介绍了，`Go`支持泛型函数和泛型类型，首先我们看一下泛型函数，下面是一个标准的泛型函数标准模板：

```go
// GenericFunc 一个标准的泛型函数模板
func GenericFunc[T any](args T) {
	  // logic code
}
```
函数签名里面多了`[T any]`部分，这就是`Go`泛型的参数列表，其中`T`就是参数，`any`为参数的约束。

![一个泛型函数](https://tva1.sinaimg.cn/large/e6c9d24egy1h0c0ok6eghj218p0u0tbo.jpg)

跑起来这个泛型函数，可以正常运行，但是别急，我们写一个泛型加法函数试试：

![可以看到报错了](https://tva1.sinaimg.cn/large/e6c9d24egy1h0c1c2xiqhj21520b0mxx.jpg)

这是为什么呢？可以看到提示说：`Invalid operation: a + b (the operator + is not defined on []T)`，这是因为我们的泛型声明`T`的类型约束在下面的逻辑代码里面进行数值运算符操作，如果大家写过`Java`里面的泛型都知道如果做数值比较操作，那我们的泛型类型参数还要写成`<T extends Comparable>`才能正常工作，这就是对不能进行数值运算符操作的类型进行规避操作，同理`Go`里面也是一样的。

为此`Go`语言在泛型中引入一个叫`类型集合`概念，下面我们改造一下代码：

```go
// 约束参数类型只能为数值类型
func add[T int64 | float64](a, b T) T {
	  return a + b
}

func main() {
    // 1 + 2 = 3
    fmt.Println("1 + 2 =",add[int64](1, 2))
    // 6.1 + 0.5 = 6.6
    fmt.Println("6.1 + 0.5 =",add[float64](6.1, 0.5))
}
```
可以看到上面可以正常运行得到正确的结果，但是有一个问题如果我们是通过内置的数据取一个类型别名怎么办？也就是以前我通过`type xx int8`这样的代码，泛型该如何限制呢？官方泛型里面映入一个`~`内置符号，这个符号会限制泛型参数底层是基于某种类型实现的变体或者别名，例如下面我这段代码：

```go
type MyInt int8

// 注意看~int8
func add[T int64 | float64 | ~int8](a, b T) T {
	  return a + b
}

func main() {
    // 限制底层数据类型
    fmt.Println("MyInt 1 + 2 = ",add[MyInt](1,2))
}
```

另外一种就是特殊比较约束，也就是上面我所的`Java`里面的`<T extends Comparable>`，比如有时候我们需要限制某个参数是否可以比较或者支持某特征某个行为，例如可比较`comparable`关键字：

```go
func SumNumbers[K comparable, V Number](m map[K]V) V {
    var s V
    for _, v := range m {
        s += v
    }
    return s
}

func whoisMin[T Number](a, b T) T {
    if a < b {
       return a
    }
    return b
}

func main() {
    // Initialize a map for the integer values
    ints := map[string]int64{
        "first":  34,
        "second": 12,
    }

    // Initialize a map for the float values
    floats := map[int8]float64{
        -128: 35.98,
        127:  26.99,
    }

    fmt.Printf("Generic Sums with Constraint: %v and %v\n",
        SumNumbers(ints),
        SumNumbers(floats))

    fmt.Println(whoisMin[int64](100, 1000))
}
```
上面的`comparable`关键字就可以限制`map`是否为可比较的类型了，说到这里我估计过不了多久某些卷王肯定又要跑去分析一下`comparable`底层实现了，我其实挺反感这样的卷王的，我不知道某些没事做的就跑去分析一下底层代码实现卷王有什么优越感？？？除了面试八股一下，难到你要去实现`GO`编译器吗？

不用分析了，我顺带说一下，其实`comparable`底层实现也就是基于一个`interface`的，只是里面有一个行为方法而已，`Go`内置的可比较数据类型在更新之后会默认实现`comparable`，这个如果大家使用其他编程语言的泛型，想一想也就知道怎么实现了，好了本篇关于`go`泛型文章就写到这里了。