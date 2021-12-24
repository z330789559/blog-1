---
title: "Bigcache Golang Garbage Collection"
date: 2021-12-24T20:39:56+08:00
---

## 为什么有这篇文章

某一天在群里摸鱼的时候，看到群里有人问`go map`的空间回收问题，把截图贴上吧：
![一个群友说map的value不是真正意义上删除](https://tva1.sinaimg.cn/large/008i3skNgy1gxcbagpu0lj30nw15e42j.jpg)

其实一位群友发出的问题引起了我注意，他的问题是：`go`的`map`的值调用了`delete`函数是不是不会立即删除？当然这个问题如果研究过或者深入`go`的内存分配或者说有了解过`go`的`gc`应该知道，这个问题答案。


**关于是`分段锁`的应用和怎么去优化`gc`带来的影响，有一个开源项目`bigcache`在这方面做的比较好。**

## BigCache的设计

> 官方作者介绍：快速，并发，基于内存的缓存库，由于是嵌入式库也省去了网络上的开销，完全基于本地内存，能保存大量数据项的同时并且对`go`语言的`garbage collection`进行了优化。

对并发锁的颗粒度减小，并且对`gc`优化它怎么做的？

1. 分段锁
2. 数据二进制存储，避免让`gc`去嵌套扫描
3. 加速并发访问
4. 避免高额的`GC`开销

官方在他们`blog`上列出了，他们当时写这个库的需求：

- 处理`10k rps` (写5000，读5000)
- `cache`对象至少存活`10`分钟
- 更快的响应时间
- `POST`请求的每条 `JSON` 消息,一有含有`ID`，二不大于`500`字节
- `POST`请求添加缓存后，`GET`能获取到最新结果



**Go map 的问题**

我在网上看到资料有提到一个问题：[https://github.com/golang/go/issues/9477](https://github.com/golang/go/issues/9477)

![问题](https://tva1.sinaimg.cn/large/008i3skNgy1gxc8j0mmoxj312u0u0n2y.jpg)

大致看了一下这个说了问题就是，`Go 1.3`和`1.4RC1`垃圾回收在扫描一个大的`map`时需要`50-70ms`时间，问官方怎么能有什么办法减小这个时间。

然后`1.5`版本，如果 `map` 的 `key` 或 `value` 中都不含指针， `GC` 便会忽略这个 `map`。


**主要的也就两个结构体`cache`和`cacheShard`:**

```go
type cache struct {
    shards []*cacheShard // 块map 解决并发锁颗粒度问题
    hash   fnv64a // 哈希函数
}
```
源码中的哈希函数用的是`fnv64a`算法，这个算法的好处是采用位运算的方式在栈上进行运算，避免在堆上分配。

![](https://tva1.sinaimg.cn/large/008i3skNgy1gxc98y338qj31et0u0dkf.jpg)

`shards`用来保存`cacheShard`的，而`cacheShard`也就是具体存储数据的缓存块，`hash`会对输入的`key`进行计算得到`cacheShard`坐标拿到具体的`cacheShard`，每个`cacheShard`都有自己的锁所以才能保证小锁颗粒度。

```go
type cacheShard struct {
    hashmap     map[uint64]uint32	// 外部索引
    entries     queue.BytesQueue // 存储序列化也就是byte形式存储的数据
    lock        sync.RWMutex // 单个锁
    entryBuffer []byte
    onRemove    onRemoveCallback // 移除回调函数

    isVerbose    bool
    statsEnabled bool
    logger       Logger
    clock        clock
    lifeWindow   uint64

    hashmapStats map[uint64]uint32
    stats        Stats
}
```
`cacheShard` 中的`hashmap`结构的类型`map[uint64]uint32`，完全和`key`使用的`string` 扯不上关系，有经验的老司机一眼就看出这其实是个索引，`uint64`存储的外部键的哈希值，而后面的`uint32`用处是存储值的位置，而真正的数据存储在`BytesQueue`中，通过外部索引的值`uint32`类型来获取里面的值所在的位置索引，而`BytesQueue`又是经过二进制序列化的数据，取的时候提供外部索引得到坐标取值，最为妙处的设计就是外部索引不存储指针，也不存储符合结构，使得 `garbage collection`的影响降到最小。


![BytesQueue](https://tva1.sinaimg.cn/large/008i3skNgy1gxc9jgne0mj32580s4adf.jpg)

`BytesQueue`结构体中的`array`是存储主要数据的`byte`数组，`capacity`是使用容量，`maxCapacity`在创建的时候可以指定最大容量，`tail`类似链表里面的尾指针，下次插入值可以通过这个指针找到插入的位置，`count`当前存储的数据条目数，`headerBuffer`是起到了一个切片发送拷贝时充当零时缓冲区的作用。

有兴趣自己去看源代码吧：[https://github.com/allegro/bigcache](https://github.com/allegro/bigcache)

## 总结 
`BigCache`的设计真的很妙，当然这个妙首先你得了解`go的gc`一些工作方式，然后针对这个这些特定，去优化数据结构，`BigCache`为了减小锁的颗粒度使用了分段分片，然后防止`gc`对数据进行扫描的耗时，有把数据采用无指针嵌套的二进制方式去存储，能避免`gc`去针对底层的数据进行引用存活相关的检测耗时，唯一缺点就是`BigCache`数据不能持久化存储。


