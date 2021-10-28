---
title: "Redis Current Limiting"
date: 2021-10-28T21:41:23+08:00
---
![](https://tva1.sinaimg.cn/large/008i3skNgy1gvvcuzx8kzj312w0lk0tt.jpg)

## Current Limiting
在编写系统时候有时候我们的系统在设计的时候就已经估算到了最大请求负载了，如果大量的请求超过系统所能承受着的值时，那么系统可能随时挂掉，所有聪明程序员就想到了`请求限流`来控制系统的可用和稳定性。

## 滑动窗口限流

滑动窗口算法将一个大的时间窗口分成多个小窗口，每次大窗口向后滑动一个小窗口，并保证大的窗口内流量不会超出最大值，这种实现比固定窗口的流量曲线更加平滑。

![滑动窗口](https://tva1.sinaimg.cn/large/008i3skNgy1gvsyv4m7k2j30y80eowff.jpg)

以系统限制用户行为为例子，比如一秒内进行某个操作`5`次，这种行为应该进行限制，滑动窗口就是记录一个滑动的时间窗口内的操作次数，操作次数超过阈值则进行限流。

```java
public boolean isActionAllowed(String userId, String actionKey, int period, int maxCount) {
    // 生成唯一的key
    String key = String.format("hist:%s:%s", userId, actionKey);
    long nowTs = System.currentTimeMillis();
    // 使用管道
    Pipeline pipe = jedis.pipelined();
    pipe.multi();
    // 添加当前操作当zset中
    pipe.zadd(key, nowTs, "" + nowTs);
    // 整理zset，删除时间窗口外的数据 符合条件的保留
    pipe.zremrangeByScore(key, 0, nowTs - period * 1000);
    // 目前窗口内有多少个操作
    Response<Long> count = pipe.zcard(key);
    // 在生命周期上加一
    pipe.expire(key, period + 1);
    pipe.exec();
    pipe.close();
    
    // 是否达到阀值
    return count.get() <= maxCount;
}

```

![过程](https://tva1.sinaimg.cn/large/008i3skNgy1gvv09e9s36j30dq0fm3yy.jpg)

## 漏捅算法
露桶算法的核心思想，就把请求进行缓冲，防止大规模数量积的请求同时冲垮系统，如下图：

![流程](https://tva1.sinaimg.cn/large/008i3skNgy1gvv0r9kdrvj309d0e8jrr.jpg)

每外来的请求都会被放到一个缓冲区也就是桶里面，如果缓冲区已经满了，那么新来的请求可能被拒绝，而缓冲区里面的请求则在符合系统设计负载的速率下进行放出，类似于一个队列一样。

在`redis`中官方没有提供相关的实现而是通过第三方的插件进行完成这个算法的，`redis-cell`是`4.0`版本提供一个限量模块，作者是来自`redis labs`的`itamar haber`，使用的`rust`语言编写的，使用很简单就API call事情，有兴趣读者可以去看看，本文就水了一篇，溜了。。。

```shell
CL.THROTTLE user123 15 30 60 1
               ▲     ▲  ▲  ▲ ▲
               |     |  |  | └───── apply 1 token (default if omitted)
               |     |  └──┴─────── 30 tokens / 60 seconds
               |     └───────────── 15 max_burst
               └─────────────────── key "user123"

```


如果要准确来说`redis-cell`的实现是`令牌桶`而非是`漏桶算法`，还是有差距的！！！！
##  其他
- https://github.com/brandur/redis-cell
