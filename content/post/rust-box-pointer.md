---
title: "Rust Box Pointer"
date: 2021-09-16T22:46:21+08:00
---

## 什么是智能指针？

在`Rust`中智能指针有很多种，用大白话说就是`一个数据结构，实现了一些特殊的trait`从而达到某种特性和功能，然后去管理某块内存上的数据，相当于一个盒子一样包装一层，这篇文章将介绍一下`Box`。

![Box](https://tva1.sinaimg.cn/large/008i3skNgy1gtr0e51tyqj619i0p2gma02.jpg)

## Deref

`Deref`这个`trait`只要实现了它那么你自定义的结构体，就可以使用`*`解引用拿取你结构体里面所包装的数据，熟悉`Rust`都知道`*`是用来解引用的，下面的`I32Box(i32)`就是实现了`Deref`从而达到直接使用`*`拿取出内部的数据。

```rust
use std::ops::Deref;

struct I32Box(i32);


// impl Deref
impl Deref for I32Box {
    type Target = i32;
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

impl I32Box {
    fn new(v:i32) -> Self {
        I32Box(v)
    }
}

fn main() {
    let n = I32Box::new(1024_i32);
    println!("{}",*n);
}
```

## Drop

`Rust`里面有`drop`函数，也有一个`Drop`的`trait`同样如果自定义的结构体实现这个`Drop`，那么就你变量离开作用域的时候执行自定义`Drop`的方法的逻辑。


```rust
impl Drop for I32Box {
    fn drop(&mut self) {
        println!("啊，挂了，I32Box({:?})被清理了！",self.0)
    }
}
```

例如为`I32Box`实现`Drop`的`drop`函数，运行：

```bash
   Compiling playground v0.0.1 (/playground)
    Finished dev [unoptimized + debuginfo] target(s) in 1.33s
     Running `target/debug/playground`


1024
啊，挂了，I32Box(1024)被清理了！
```
当`I32Box`离开自己的作用域那么它会自动执行`drop`方法。

## Box

有了这些特殊的`trait`基础之后，智能指针就自身有了这种行为了，就能管理内部的数据了。

`Box`智能指针它管理的数据，内存是分配在`heap`上的，可以在堆上存储我们自定义的的数据，而把指针放在栈上。有这些特性，我们就可以使用`Box`实现一个类似动态数组的数据结构。

```rust
#[derive(Debug)]
struct DynamicArr {
    cap:i32,
    len:i32,
    data:Box<[i32]>,
}

impl DynamicArr {

    fn new() -> DynamicArr {
        DynamicArr{
            cap:10,
            len:0,
            data: Box::new([0;10]),
        }
    }
    
    fn push(&mut self,v:i32) {
        // expansion
        if self.len == self.cap {
            // add 10 capacity
            let mut new_arr = Box::new([0;20]);
            let mut index = 0;
            for v in self.data.iter() {
                new_arr[index] = *v;
                index += 1;
            }
            self.data = new_arr;
            self.cap += index as i32;
        }
        self.data[self.len as usize] = v;
        self.len+=1;
    }
}



fn main() {
    let mut arr = DynamicArr::new();
    
    for v in 0..16 {
        arr.push(v);
    }

    println!("{:#?}",arr)
}
```

例如上面通过`Box`智能指针实现一个自定义`DynamicArr`结构，它在我们添加数组的时候会检查容量，当达到阈值就会发生`expansion`，上面只是个小`demo`，还有`bug`，只是为了演示`Box`智能指针的使用。

## 其他

- [I32Box](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=47cacd3f18ff2cb8361a40555dfc7d73)
- [DynamicArr](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=fdfeea95c1ecae0b8286471e320fd331)
