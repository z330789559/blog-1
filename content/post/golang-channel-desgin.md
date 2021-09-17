---
title: "Golang Channel Desgin"
date: 2021-09-16T22:19:16+08:00
---

相信大家在开发的过程中经常会使用到`go`中并发利器`channel`，`channel` 是`CSP`并发模型中最重要的一个组件，两个独立的并发实体通过共享的通讯`channel`进行通信。大多数人只是会用这么个结构很少有人讨论它底层实现，这篇文章讲写写`channel`的底层实现。


## channel
`channel`的底层实现是一个结构体，[源代码](https://github.com/golang/go/blob/41d8e61a6b9d8f9db912626eb2bbc535e929fefc/src/runtime/chan.go#L32)如下:
```go
type hchan struct {
    qcount   uint           // total data in the queue
    dataqsiz uint           // size of the circular queue
    buf      unsafe.Pointer // points to an array of dataqsiz elements
    elemsize uint16
    closed   uint32
    elemtype *_type // element type
    sendx    uint   // send index
    recvx    uint   // receive index
    recvq    waitq  // list of recv waiters
    sendq    waitq  // list of send waiters

    // lock protects all fields in hchan, as well as several
    // fields in sudogs blocked on this channel.
    //
    // Do not change another G's status while holding this lock
    // (in particular, do not ready a G), as this can deadlock
    // with stack shrinking.
    lock mutex
}
```

可能看源代码不是很好看得懂，这里我个人画了一张图方便大家查看，我在上面标注了不同颜色，并且注释其作用。

![hchan struct](https://tva1.sinaimg.cn/large/008i3skNgy1gtkvnaup23j615r0u0mzv02.jpg)

通道像一个`传送带`或者`队列`，总是遵循`FIFO`的规则，保证收发数据的顺序，通道是`goroutine`间重要通信的方式，是并发安全的。


## buf

`hchan`结构体中的`buf`指向一个循环队列，用来实现循环队列，`sendx`是循环队列的队尾指针，`recvx`是循环队列的队头指针，`dataqsize`是缓存型通道的大小，`qcount`是记录通道内元素个数。


在日常开发过程中用的最多就是`ch := make(chan int, 10)`这样的方式创建一个通道，如果这要声明初始化的话，这个通道就是有缓冲区的，也是图上紫色的`buf`，`buf`是在`make`的时候程序创建的，它有`元素大小*元素个数`组成一个循环队列，可以看做成一个环形结构，`buf`则是一个指针指向这个环。

![ring](https://tva1.sinaimg.cn/large/008i3skNgy1gtl27ej1w4j60qo0k0t9202.jpg)

上图对应的代码那就是`ch = make(chan int,6)`，`buf`指向这个环在`heap`上的地址。

```go
func makechan(t *chantype, size int) *hchan {
    elem := t.elem

    // compiler checks this but be safe.
    if elem.size >= 1<<16 {
        throw("makechan: invalid channel element type")
    }
    if hchanSize%maxAlign != 0 || elem.align > maxAlign {
        throw("makechan: bad alignment")
    }

	  mem, overflow := math.MulUintptr(elem.size, uintptr(size))
    if overflow || mem > maxAlloc-hchanSize || size < 0 {
        panic(plainError("makechan: size out of range"))
    }

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
    // buf points into the same allocation, elemtype is persistent.
    // SudoG's are referenced from their owning thread so they can't be collected.
    // TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
    var c *hchan
    switch {
    case mem == 0:
        // Queue or element size is zero.
        c = (*hchan)(mallocgc(hchanSize, nil, true))
        // Race detector uses this location for synchronization.
      c.buf = c.raceaddr()
    case elem.ptrdata == 0:
        // Elements do not contain pointers.
        // Allocate hchan and buf in one call.
        c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
        c.buf = add(unsafe.Pointer(c), hchanSize)
    default:
        // Elements contain pointers.
        c = new(hchan)
        c.buf = mallocgc(mem, elem, true)
    }

    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size)
    lockInit(&c.lock, lockRankHchan)

    if debugChan {
        print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
    }
    return c
}
```
上面就是对应的代码实现，上来它会检查你一系列参数是否合法，然后在通过`mallocgc`在内存开辟这块空间，然后返回。

## sendx and recvx
下面我手动模拟一个`ring`实现的代码：
```go
// Queue cycle buffer
type CycleQueue struct {
    data                  []interface{} // 存放元素的数组，准确来说是切片
    frontIndex, rearIndex int           // frontIndex 头指针,rearIndex 尾指针
    size                  int           // circular 的大小
}

// NewQueue Circular Queue
func NewQueue(size int) (*CycleQueue, error) {
    if size <= 0 || size < 10 {
      return nil, fmt.Errorf("initialize circular queue size fail,%d not legal,size >= 10", size)
    }
    cq := new(CycleQueue)
    cq.data = make([]interface{}, size)
    cq.size = size
    return cq, nil
}

// Push  add data to queue
func (q *CycleQueue) Push(value interface{}) error {
    if (q.rearIndex+1)%cap(q.data) == q.frontIndex {
        return errors.New("circular queue full")
    }
    q.data[q.rearIndex] = value
    q.rearIndex = (q.rearIndex + 1) % cap(q.data)
    return nil
}

// Pop return queue a front element
func (q *CycleQueue) Pop() interface{} {
    if q.rearIndex == q.frontIndex {
        return nil
    }
    v := q.data[q.frontIndex]
    q.data[q.frontIndex] = nil // 拿除元素 位置就设置为空
    q.frontIndex = (q.frontIndex + 1) % cap(q.data)
    return v
}
```

循环队列一般使用空余单元法来解决队空和队满时候都存在`font=rear`带来的二义性问题，但这样会浪费一个单元。`golang`的`channel`中是通过增加`qcount`字段记录队列长度来解决二义性，一方面不会浪费一个存储单元，另一方面当使用`len`函数查看队列长度时候，可以直接返回`qcount`字段，一举两得。

![对应的源代码](https://tva1.sinaimg.cn/large/008i3skNgy1gtl2srriq6j60sf0k4tad02.jpg)

当我们需要读取的数据的时候直接从`recvx`指针上的元素取，而写就从`sendx`位置写入元素，如图：

![读写指针](https://tva1.sinaimg.cn/large/008i3skNgy1gtl2nxgw4vj60qo0k0wew02.jpg)

## sendq and recvq

当写入数据的如果缓冲区已经满或者读取的缓冲区已经没有数据的时候，就会发生协程阻塞。

![send](https://tva1.sinaimg.cn/large/008i3skNgy1gtl38rio4tj60va0u0dj902.jpg)

如果写阻塞的时候会把当前的协程加入到`sendq`的队列中，直到有一个`recvq`发起了一个读取的操作，那么写的队列就会被程序唤醒进行工作。

![全部挂起态](https://tva1.sinaimg.cn/large/008i3skNgy1gtl3no33luj60qo0k0q3v02.jpg)

当缓冲区满了所有的`g-w`则被加入`sendq`队列等待`g-r`有操作就被唤醒`g-w`，继续工作，这种设计和操作系统的里面`thread`的`5`种状态很接近了，可以看出`go`的设计者在可能参考过操作系统的`thread`设计。

当然上面只是我简述整个个过程，实际上`go`还做了其他细节优化，`sendq`不为空的时候，并且没有缓冲区，也就是无缓冲区通道，此时会从`sendq`第一个协程中拿取数据，有兴趣的`gopher`可以去自己查看源代码，本文也是最近笔者在看到这块源代码的笔记总结。


![](https://tva1.sinaimg.cn/large/008i3skNgy1gu091pmsx9j61bi0hcabs02.jpg)