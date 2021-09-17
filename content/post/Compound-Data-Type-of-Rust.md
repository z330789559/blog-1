---
title: "Compound Data Type of Rust"
date: 2021-09-16T22:25:30+08:00
---

## 概 述
> 好久没有更新`rust`相关的内容了，更新一波`Rust`的内容，本篇讲介绍一下`Rust`中的复合数据类型。

## Composite Type

`复合数据类型`是一种数据类型，它可以原始的基本数据类型和其它的复合类型所构成， 构成一个复合类型的动作，又称作组合。

本文讲介绍一下在`Rust`中有`tuple`、`array`、`struct`、`enum`几个复合类型。


## tuple

`tuple`即`元组`，元组类型是由多个不同类型的元素组成的复合类型，通过`()`小括号把元素组织在一起成一个新的数据类型。元组的长度在定义的时候就已经是固定的了，不能修改，如果指定了元素的数据类型，那么你的元素就要对号入座！！！否则编译器会教训你！




**例子：**

```rust
fn main() {
    // 指定数据类型
    let tup_type:(i8,i32,bool) = (21,-1024,true);
    // 解构元素
    let (one,two,three) = tup_type;
    // 二维的元组
    let tup_2d:(f64,(i8,i32,bool)) = (3.1415927,(one,two,three));
    println!("tup_2d = {:?}",tup_2d);
    // 索引
    println!("π = {:?}",tup_2d.0);
}
```

元组的访问方式有好几种，通过下标去访问，也可以使用解构赋值给新的变量去访问，但是不支持迭代器去访问。

```rust
for v in tup_2d.1.iter() {
        println!("{}",v)
}
```

```bash
   Compiling playground v0.0.1 (/playground)
error[E0599]: no method named `iter` found for tuple `(i8, i32, bool)` in the current scope
  --> src/main.rs:10:23
   |
10 |     for v in tup_type.iter() {
   |                       ^^^^ method not found in `(i8, i32, bool)`

error: aborting due to previous error

For more information about this error, try `rustc --explain E0599`.
error: could not compile `playground`

To learn more, run the command again with --verbose.
```

`元组`的每个元素的类型可以不同，因此您无法对其进行迭代。元组甚至不能保证以与类型定义相同的顺序存储数据，因此即使您自己为它们实现`Iterator`，它们也不适合进行有效的迭代。

但是如果元素是支持实现了`Iterator`就可以通过`.iter()`进行迭代访问。

```rust
    let mut arrays:[usize;5] = [0;5];
    
    for i in 0..5 {
        arrays[i] = i+1;
    }
    
    println!("{:?}",arrays);
    
    let tup_arr:(&str,[usize;5]) = ("tup_arr",arrays);
    
    println!("{:?}",tup_arr);
    
    for v in tup_arr.1.iter() {
        println!("{}",v)
    }
```
例如上的元素是一个`array`，`Rust`中的数组和其他语言一样，一组类型相同的元素组成的复合类型，数组在底层存储是一块连续的内存空间。

## array

`Rust`中的数组声明是`[T;n]`进行的，`T`是元素类型，`n`是这组元素有多少个坑位，创建的时候可以去掉类型和大小，程序会自动推断出来。

```rust
    // 数组
    let arr:[f32;3] = [1.0,2.2,3.33];
    
    println!("{:?}",arr);
    
    // 类型自动推导
    let arr_infer = ["Hello",",","World!"];
    
    let mut str = String::new();
    // 迭代器
    for v in arr_infer.iter() {
        str.push_str(v);
    }
    
    println!("str = {}",str);
```

[点击查看元组代码案例](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=195d35086371375f182fc67922d81b44)

## enum

枚举类型，如果你之前从事过`Java`相关的开发应该不陌生，在`Rust`里面也有`枚举`类型，枚举类型是一个自定义数据类型，通过`enum`关键字来声明，`body`里面可以包含多个自定义的枚举值，枚举可以用来限制某个值或者类型范围。

```rust
    #[derive(Debug)]
    enum Gender {
        Boy,
        Girl,
    }
```
上面就定义一了个类型名字为`Gender`的枚举，`Boy`和`Girl`是枚举可供使用的值，`#[derive(Debug)]`注释是让`Gender`自动实现`Debug tarit`后面文章将深入。

## struct

结构体可以把一些自定义的数据类型通过已有的类型组装成一个新的自定义数据类型，通过`struct`关键字就可以创建一个结构体，结构体字段格式`name:type`，`name`是结构体字段名，`type`是字段的类型，默认是不可变的。

```rust
fn main() {

    // 枚举现在取值范围
    #[derive(Debug)]
    enum Gender {
        Boy,
        Girl,
    }
    
    // 定义一个结构体
    #[derive(Debug)]
    struct Programmer<'skill> {
        name: String,
        skill: [&'skill str; 3],
        sex: Gender,
    }
    
    // 创建一个实例
    let engineer = Programmer {
        name: String::from("Jaco Ding"), // String类型内容可变
        skill: ["Java","Go","Rust"], // 一个长度为3的字符串面量类型的数组
        sex:Gender::Boy, // 通过枚举限制参数类型
    };
    
    println!("engineer = {:?}",engineer);
}
```
有了自定义的类型了也就是`struct`就可以通过自定义的类型来处理一些特殊的需求了，例如下面的代码定义了一个元素类型为`Programmer`长度为`2`的数组。

```rust
    let Doris = Programmer {
        name: String::from("Doris"),
        skill: ["Vue","TypeScript","JavaScript"],
        sex:Gender::Girl,
    };
    
    let Jaco = Programmer {
        name: String::from("Jaco"),
        skill: ["Java","Go","Rust"],
        sex:Gender::Boy,
    };
    
    let employees:[Programmer;2] = [Doris,Jaco];
    
    for e in employees.iter() {
        println!("{:?}",e)
    }
```

结构体`Programmer`上的`<'skill>` 解决`skill`数组的生命周期问题`undeclared lifetime`，所有权问题，所有权是`Rust`语言核心的知识点，这些在后面文章中慢慢更新。

## 小结

`Rust`中的结构体还有两种特殊结构：`元组结构体`、`单结构体`，枚举也有`带有参数`的枚举。。。本文就学习总结了一下常见复合类型的使用，未深入。
