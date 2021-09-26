---
title: "Go Slice Expansion Mechanism"
date: 2021-09-26T23:06:08+08:00
---
切片是`Golang`里面一个复合数据类型，可以把它看做为一个可变长度的数组，和`动态数组`一样，在创建的时候我们可以指定容量大小，如果不够了，它还能指定扩容，基本的`crud`没有什么可说的本篇文章将写写切片底层的实现。


## slice struct

其实`slice`在底层就是一个`struct`声明一个结构体，结构如下：

```go
type slice struct {
    array unsafe.Pointer
    len int
    cap int
}
```

![内存结构体](https://tva1.sinaimg.cn/large/008i3skNgy1guu9lfrp8tj60kt070q3f02.jpg)


`slice`本身一个结构体里面包含了`array`和`len`和`cap`成员变量，`array`是一个指针指向真正存储数据的内存头元素地址，`len`记录着当前实际的元素个数，`cap`记录当前切片的容量，如上图所示。



## append expansion


在说`append`扩容机制之前，先看一个下面的题目，最终输出什么？？

```go
package main

import (
	"fmt"
)

func main() {
    array := [...]int{1,2,3,4,5}
    s1 := append(array[:3],array[4:]...)
    s2 := s1
    s1 = append(s1,[]int{6,7}...)
    test(s2)
    test(s1)
    fmt.Println(s1,s2) 
}

func test(s []int) {
    s = append(s,10)
    for i := range s {
        s[i]*=2
    }
}
```
估计大多数人肯定会秒回答说`[1 2 3 5 5 6 7] [1 2 3 5 5]`，如果回答这个就说明你连`go`里面函数参数是`值传递`还是`引用传递`都没有搞清楚，而且你对`append`也没有搞清楚。

上面代码运行正确的答案是`[2 4 6 10 12 14] [2 4 6 10]`！

为什么会这样？这个首先在`Go`语言中的所有东西都是以值传递的，也就是说，一个函数总是得到一个被传递的东西的副本，就像有一个赋值语句将值赋给参数一样。


![append expansion](https://tva1.sinaimg.cn/large/008i3skNgy1guubusz0luj60kp09i0t502.jpg)

上图就是整个`append`扩容过程，首先检测原始的`array`底层的那个数组容量够不够，如果不够就会先申请一块新的内存，当然这个扩容大小有相应的策略的，我画的这幅方便理解，所以我这里就当原始数量的`*2`进行扩容。扩容之后把旧切片的元素复制到新的切片中，然后将相应的添加的元素追加进去即可。


了解这些就很方便理解为什么？？上面结果是`[2 4 6 10 12 14] [2 4 6 10]`了，首先我将`s1`赋值给了`s2`，`s2`和`s1`共用一块底层的元素内存，然后将`s1`进行了`append`操作，这一操作会触发扩容机制，所有在扩容之后`s2`和`s1`底层数组完全是不一样的！`test(s []int)`又是值拷贝，拷贝也是底层的指针，所有操作还是一块内存，就会影响到外面的，这里的拷贝只会影响到`unsafe.Pointer`指针类型，`len`和`cap`则是值拷贝。
