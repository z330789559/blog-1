---
title: "Rust Lifetime"
date: 2021-09-16T22:47:17+08:00
---

## lifetime寿命
`Rust`中的每一个引用都有一个有效的作用域，生命周期就是为这个作用域服务的，大部分生命周期编译器可以推断出来，可以是隐式的。但是如果在某些情况下编译器就无法正常推断出来了，需要我们自己手动标注，标注生命周期语法就是`'a`这样的语法。

## 为什么需要生命周期？

例如下面例子就是在两个字符串切片里面查找最长的那个并且返回！

```rust linenums='1'
// 'a 是指3个引用的作用域生命周期要一致
fn find_long_str<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```
上面我就加注了生命周期标识符，如果不加编译器会报错，原因是因为我们这个函数引用的是外部的变量，不能确定引用的变量是否已经被销毁了，那这样就是悬垂引用！

```rust linenums='1'
    let str1 = String::from("Hello");
    let str2;
    let result;
    {
        //let str2 = String::from(" World!");
        str2 = String::from(" World!");
        result = find_long_str(str1.as_str(), &str2);
    }
    // 这里借用检测就提示 引用了已经销毁的资源了
    println!("{}", result);
```

![大致过程图](https://tva1.sinaimg.cn/large/008i3skNgy1gt68o41m7aj30po0g0q3r.jpg)

加了生命周期标识符之后，如果我把`let str2 = String::from(" World!");`取消注释放在一个内部作用域里面定义，那么这时调用
`find_long_str`编译器就会报错，因为我在下面出了作用域还使用了`find_long_str`返回的结果，而这个结果可能就是`str2`的内容，
使用这个是违反了所有权规则的，`str2`离开内部作用域就被销毁了。

在标注生命周期`fn find_long_str<'a>(x: &'a str, y: &'a str) -> &'a str`之后编译器就知道输入参数和返回参数生命周期是要一致的，并且返回值生命周期肯定是取生命周期最短的那个的。

## 总结
- 生命周期是确保被引用的值是有效的。
- 引用的生命周期肯定是小于或者等于资源所有者的。
- 如果是在函数里面创建的资源，应该是直接返回其所有权，而不是引用。
- 每个生命周期标注都有不同的生命周期，如果有输入的生命周期，那么输出的生命周期也是一致的。
- `self`的生命周期会被赋给输出的生命周期。

## 其他
- 当然上面是我刚刚入坑总结话，有错误地方望大佬指教！
- [https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=a868aa030fa934b22cd770727f42724d](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=a868aa030fa934b22cd770727f42724d)