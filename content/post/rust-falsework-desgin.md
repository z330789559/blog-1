---
title: "Rust Falsework Desgin"
date: 2021-09-16T22:22:45+08:00
---

## 前言

>断断续续看了半个月的`Rust`书，学了点基础，不知道做什么好？？于是我想着用业余时间撸一个`command line application`的骨架。然后就有了`https://github.com/auula/falsework`这个项目，`falsework`可以帮助你快速构建一个命令行应用。这篇文章写写`falsework`的使用并在后部分介绍一下怎么实现的。


![rust学习曲线](https://tva1.sinaimg.cn/large/008i3skNgy1gtta3lv37zj617x0u040n02.jpg)

看了看这张图，那我估计还在`坡道起步`。。

## 导入依赖

- https://crates.io/crates/falsework

在你的项目中添加依赖如下：

```toml
[dependencies]
falsework = "0.1.3"
```


## 快速构建

```rust
use std::error::Error;
use falsework::{app, cmd};

fn main() {
    
    // 通过falsework创建一个骨架
    let mut app = falsework::app::new();
    
    
    // 应用元数据信息
    app.name("calculator")
        .author("Leon Ding <ding@ibyte.me>")
        .version("0.0.1")
        .description("A calculator that only supports addition.");

    // 构建命令行项
    let mut command = cmd::CommandItem {
        // run 命令所对应命令行逻辑代码
        run: |ctx| -> Result<(), Box<dyn Error>> {
            // 通过上下文获取flag绑定的数据
            let x = ctx.value_of("--x").parse::<i32>().unwrap();
            let y = ctx.value_of("--y").parse::<i32>().unwrap();
            println!("{} + {} = {}", x, y, x + y);
            // 如果处理发生了错误则调用 cmd::err_msg 会优雅的退出
            // Err(cmd::err_msg("Application produce error！"));
            Ok(())
        },
        // 命令帮助信息
        long: "这是一个加法计算程序需要两个flag参数 --x --y",
        // 命令介绍
        short: "加法计算",
        // 通过add激活命令
        r#use: "add",
    }.build();
    
    // 给add命令绑定flag
    command.bound_flag("--x", "加数");
    command.bound_flag("--y", "被加数");
    
    // 往app里面添加一个命令集
    app.add_cmd(command);
    
    // 最后run 程序开始监听命令行输入
    app.run();
}
```

上面这个例子运行起来结果:

```shell
$: ./calculator add --x=10 --y=10
10 + 10 = 20
```

到此为止你就快速构建一个命令行计算器了，你只需要写你核心逻辑，其他操作`falsework`帮助你完成。

1. 例如如果我不记得了命令了，只记得一个单词或者字母，程序会帮助你修复：

```shell
$: ./calculator a

You need this command ?
	add
a : The corresponding command set was not found!
```
2. 可以看到程序提示你有一个对应的`add`命令可以使用，如果不知道`add`有啥参数，在后面
加上`--help`即可获得帮助信息：

```shell
$: ./calculator add --help

Help:
  这是一个加法计算程序需要两个flag参数 --x --y

Usage:
  calculator add [flags]

Flags:
  --y, 被加数
  --x, 加数
```
构建出来主程序预览：

```shell
$: ./calculator


A calculator that only supports addition.
calculator 0.0.1 Leon Ding <ding@ibyte.me>

Usage:
  calculator  [command]

Available Commands:
  add	加法计算

Flags:
  --help   help for calculator

Use "calculator [command] --help" for more information about a command.
```

## 其他操作
有多种构建方式，例如下面的：

```rust
    #[test]
    fn test_add_commands() {
        let mut app = falsework::app::new();

        app.name("calculator")
            .author("Leon Ding <ding@ibyte.me>")
            .version("0.0.2")
            .description("A command line program built with Falsework.");


        let command_list = vec![
            cmd::CommandItem {
                run: |_ctx| -> Result<(), Box<dyn Error>> {
                    // _ctx.args 获取命令行参数
                    println!("call foo command.");
                    Ok(())
                },
                long: "这是一个测试命令，使用foo将调用foo命令。",
                short: "foo命令",
                r#use: "foo",
            },
            cmd::CommandItem {
                run: |_ctx| -> Result<(), Box<dyn Error>> {
                    println!("call bar command.");
                    Ok(())
                },
                long: "这是一个测试命令，使用bar将调用bar命令。",
                short: "bar命令",
                r#use: "bar",
            },
        ].iter().map(|c| c.build()).collect();

        app.commands(command_list);

        println!("{:#?}", app);
    }
```


## Falsework的设计

![falsework结构图](https://tva1.sinaimg.cn/large/008i3skNgy1gtsagkvp4lj60qo0k0tau02.jpg)

整体来说也就是`3`大块，对应一个`cli`程序抽象的话也就这么多了，然后就是对单块的结构进行封装如图，然后就是构建一个骨架了。


![参数规格解析](https://tva1.sinaimg.cn/large/008i3skNgy1gttaacjpohj60qo0k0mxd02.jpg)

拿上面这个例子来说，在你敲击回车那一刻，骨架要帮做很多`args`解析的事情，然后构建对应任务处理。

![逻辑图](https://tva1.sinaimg.cn/large/008i3skNgy1gttac10fbvj60r00k6dgc02.jpg)

这张图就是当程序监听到命令行有输入操作的时候要做的事情，我把他抽象画成了一个图，大部分程序员应该能看得懂，看不懂可以对照仓库代码看看，虽然代码写的一坨，很`low`代码，毕竟还只是刚刚入坑`Rust`。。。。新手朋友可以看看，对了有启发或者帮助记得留一个`star`！

## 仓库
- [https://github.com/auula/falsework](//github.com/auula/falsework) 
