---
title: "Golang Channel 学习笔记"
date: 2019-09-03T20:49:19+08:00
draft: false
categories: ["Golang", "学习笔记"]
tags: ["Golang","源码学习"]
keyword: ["Golang","源码学习", "Channel"]
slug: Golang Channel Notes
---

## 简介
Channel 是 Go 语言中并发模型的重要组成部分，可以在不同的 goroutine 之间进行数据的传输。

在日常使用中，channel 可以分为两种：

- 无缓冲 channel：在无缓冲 channel 中，数据的发送和读取操作需要同时进行，当只有一个 goroutine A 进行数据的读取（写入）时，需要等待另一个 goroutine B 通过这个 channel 进行数据写入（读取）。在此之前 goroutine A 会一直阻塞，直到 goroutine B来拯救它。
- 有缓冲 channel：在有缓冲 channel 中，会自带一个缓冲循环队列。当 goroutine 进行数据读取时，如果缓冲队列中有数据便直接读取，如果没有便阻塞等待其他 goroutine 写入数据。当 goroutine 进行数据写入时，如果缓冲队列没有满，则直接写入数据后离开。如果缓冲队列已满，则需要等待其他 goroutine 来消费缓冲队列，才能继续写入，在此之前只能一直阻塞。

## Channel 数据结构

首先，我们看一下 Go 语言中 Channel 的数据结构：

```go
# go1.13:src/runtime/chan.go:32 
type hchan struct {
	qcount   uint           // 当前channel中元素的个数
	dataqsiz uint           // 循环队列的长度
	buf      unsafe.Pointer // 指针, 指向一个长度为dataqsize的数组, 即缓冲队列
	elemsize uint16 // channel可以接收的元素大小(单个)
	closed   uint32	// channel的关闭状态
	elemtype *_type // channel可以接收的元素类型
	sendx    uint   // 标示当前可发送元素的位置(数组下标)
	recvx    uint   // 标示当前可接收元素的位置(数组下标)
	recvq    waitq  // 阻塞的接收 goroutine 队列
	sendq    waitq  // 阻塞的发送 goroutine 队列

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex	// 原子锁🔒
}

# go1.13.7:src/runtime/chan.go:53
type waitq struct {
	first *sudog	//指向队列头部的 goroutine 指针
	last  *sudog	//指向队列尾部的 goroutine 指针
}
```

channel 在 Go 语言中是以  ```hchan``` 结构体存在的，结构体中的字段如注释所示。其中  ```sudog``` 可以认为是 goroutine 的额外封装。

## Channel 创建

channel 的创建最终都会调用  ```makechan()``` 接口，话不多说先上源码：

```go
# go1.13:src/runtime/chan.go:71
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe.
	if elem.size >= 1<<16 {	// channel元素大小校验, 不能超过64KB
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign { // 内存对齐限制检查
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.size, uintptr(size)) //申请内存的预检查
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

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; elemalg=", elem.alg, "; dataqsiz=", size, "\n")
	}
	return c
}
```

首先是一系列的安全检查：

- 检查 channel 元素类型的大小，不能超过64KB。

- 检查 chennel 元素结构体是否已经对齐，以及 channel 中的元素内存对齐值不得超过  maxAlign （在当前版本中 maxAlign 的值为8）

- 检查缓冲队列的内存申请是否超过限制，主要有三个判断：

  - 申请的内存是否溢出

  - 是否超过允许创建的最大内存限制。这里需要减去 channel 结构本身的内存占用 hchanSize，这个内存占用大小的定义如下：

    ```Go
    unsafe.Sizeof(hchan{}) + uintptr(-int(unsafe.Sizeof(hchan{}))&(maxAlign-1))
    ```

  - 当然，申请的内存大小不能为负数

通过安全检查之后，会声明 ```hchan``` 指针 c，根据3种不同的情况对 c 进行初始化：

- 创建的是无缓冲 channel ：即申请缓冲队列的大小为0，这个时候会为 c 申请 hchanSize 大小的内存。同时调用 raceaddr() 对缓冲队列指针地址进行读写操作，主要作用是为了防止在调用 len() 或 cap() 函数时读取内存地址和其他操作产生资源竞争。
- 创建的是有缓冲 channel 且缓冲队列元素不包含指针：会为 c 申请一段 hchanSize + mem 大小的内存，并将缓冲队列指针指向内存地址后段，即加上 hchanSize 的偏移量后指向一段长度为 mem 的内存地址。
- 创建的是有缓冲 channel 且缓冲队列元素包含指针：分别为 c 和缓冲队列 c.buf 分配对应大小的内存。

剩下的操作便是一些上文提到的变量初始化和 Debug 日志输出了。

## Channel 发送

channel 的发送逻辑实现主要是在 ```src/runtime/chan.go``` 中的  ```chansend()``` 和 ```send()```，由于实现逻辑比较冗长，情况也比较多，这里就不直接代码骑脸，我们根据不同的情况具体分析：

### 无缓冲 channel 发送

#### 直接发送

前文说过无缓冲 channel 在发送时，如果没有对应接收的 goroutine，就会将当前的发送 goroutine 放入 sendq 队列，所以发送时，第一步会先检查 recvq 队列是否有接收 goroutine 在等待接收， 如果已经有等待接收的 goroutine，便直接调用 ```send()``` 进行数据发送：

```Go
# go1.13:src/runtime/chan.go:190
if sg := c.recvq.dequeue(); sg != nil {
	// Found a waiting receiver. We pass the value we want to send
	// directly to the receiver, bypassing the channel buffer (if any).
	send(c, sg, ep, func() { unlock(&c.lock) }, 3)
	return true
}
```

#### 发送阻塞

当接收队列为空的时候，程序会检查当前 channel 的缓冲队列是否可用（当然在无缓冲 channel 中答案时肯定不行的），于是会获取当前发送的 goroutine，根据这个 goroutine 封装一个上文提到的 ```sudog``` 结构并放入 sendq 队列中并将当前的 goroutine 设置为休眠状态（通过调用 ```goparkunlock()```）。同时也会调用 ```KeepAlive()``` 将当前发送的数据（实际上是一个指向当前数据的指针）标记为 reachable 以防止数据在发送前被释放或修改：

```Go
# go1.13:src/runtime/chan.go:219
// Block on the channel. Some receiver will complete our operation for us.
gp := getg()
mysg := acquireSudog()
mysg.releasetime = 0
if t0 != 0 {
	mysg.releasetime = -1
}
// No stack splits between assigning elem and enqueuing mysg
// on gp.waiting where copystack can find it.
mysg.elem = ep
mysg.waitlink = nil
mysg.g = gp
mysg.isSelect = false
mysg.c = c
gp.waiting = mysg
gp.param = nil
c.sendq.enqueue(mysg)
goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)
// Ensure the value being sent is kept alive until the
// receiver copies it out. The sudog has a pointer to the
// stack object, but sudogs aren't considered as roots of the
// stack tracer.
KeepAlive(ep)
```

### 有缓冲 channel 发送

#### 直接发送

同无缓冲 channel 一样，发送的第一件事是对接收队列 recvq 进行检查（因为此时如果 recvq 队列有正在等待接收的 goroutine，那么缓冲区的数据一定已经被读取完了），如果有正在等待接收的 goroutine，那么跟上文的无缓冲 channel 是一样的处理。不同的地方在于，如果 recvq 队列为空，程序会继续检查当前缓冲区是否还有空间，如果还有空间，便将发送的数据写入缓冲区，而当前的 goroutine 就可以继续往下执行了：

```Go
# go1.13:src/runtime/chan.go:197
if c.qcount < c.dataqsiz {
	// Space is available in the channel buffer. Enqueue the element to send.
	qp := chanbuf(c, c.sendx) //获取队列下一个位置发送数据的存储位置
	if raceenabled {	//竞争检测，在当前版本中 raceenabled 默认为 false，只有在运行时加上 -race 才会成为 true
		raceacquire(qp)
		racerelease(qp)
	}
	typedmemmove(c.elemtype, qp, ep)	//将发送的数据复制到缓冲区
	c.sendx++	//缓冲区的发送指针位置调整
	if c.sendx == c.dataqsiz { //循环队列常规操作
		c.sendx = 0
	}
	c.qcount++
	unlock(&c.lock)
	return true
}
```

#### 发送阻塞

如果缓冲队列容量已满，那么即使是有缓冲的 channel，也会将当前发送数据的 goroutine 挂起，具体操作和无缓冲 channel 是一致的，这里就不做赘述了，可以自行查看[无缓冲 channel 发送数据阻塞过程](#发送阻塞)。

### 唤醒

在阻塞状态进行了漫长的等待之后，有一天，一个从远方而来进行接收数据的 goroutine 终于出现，这个时候休眠状态的发送 goroutine 被唤醒，进行数据的收尾及释放：

```Go
# go1.13:src/runtime/chan.go:243
// someone woke us up.
if mysg != gp.waiting {
	throw("G waiting list is corrupted")
}
gp.waiting = nil
if gp.param == nil {
	if c.closed == 0 {
		throw("chansend: spurious wakeup")
	}
	panic(plainError("send on closed channel"))
}
gp.param = nil
if mysg.releasetime > 0 {
	blockevent(mysg.releasetime-t0, 2)
}
mysg.c = nil
releaseSudog(mysg)
```

主要的检查有两点，一个是当前 goroutine 的锁定是否被篡改：gp.waiting 指向当前 goroutine 被占用等待的  ```sudog``` 结构体。另一个检查是当前 channel 的关闭状态，倘若当前 channel 已经关闭，则会导致程序Panic，因为从一个已经关闭的 channel 中写数据是不被允许的。

检查完毕之后便是对资源的回收。至此一个完整的 channel 发送流程便走完了。

### How to send ？

上文提到如果是两个 gouroutine 直接进行数据交流（不经过缓冲队列），程序会调用 ```send()``` 进行数据发送，那么它是如何发送数据呢？

```Go
# go1.13:src/runtime/chan.go:269
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if raceenabled {	// 正常运行状态下，raceenabled 为 false， 所以这里我们无视吧🤫
		if c.dataqsiz == 0 {
			racesync(c, sg)
		} else {
			// Pretend we go through the buffer, even though
			// we copy directly. Note that we need to increment
			// the head/tail locations only when raceenabled.
			qp := chanbuf(c, c.recvx)
			raceacquire(qp)
			racerelease(qp)
			raceacquireg(sg.g, qp)
			racereleaseg(sg.g, qp)
			c.recvx++
			if c.recvx == c.dataqsiz {
				c.recvx = 0
			}
			c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
		}
	}
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep) // 直接将数据 ep 拷贝到接收的 goroutine 中
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)	//将当前的 sudog 设置为接收方 goroutine 唤醒后的参数
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
```

raceenabled 竞争检测相关这里我们就略过不展开讲了（其实是不会）。有兴趣的同学可以自行查阅，不要忘了学成归来评论分享。

在分析发送逻辑之前我们要明确两件事：一个是在 ```send()``` 中传入的 sg 已经不是发送方的  ```sudog``` 结构体而是数据接收队列中的接收方。还有一点是此时数据接收方的 goroutine 必然是休眠状态的。

在 ```send()``` 中程序调用 ```sendDirect()``` 将数据 ep 的内容<font color=red>**直接拷贝**</font>到接收 goroutine 的内存空间中，然后回收指向该内存的指针，同时也会将当前的 ```sudog``` 设置为接收方 goroutine 唤醒后的参数。最后通过调用 ```goready()``` 唤醒休眠的接收 goroutine 来完成数据传输的过程。

## Channel 接收

channel 的接收逻辑同样在 ```src/runtime/chan.go``` 中，主要也是 ```chanrecv()``` 和 ```recv()``` 两个接口。接收逻辑大体上和发送十分类似，~~甚至可以理解为仅仅是数据流动的方向改变了而已。~~主要的差异体现在数据的交换逻辑。

### 无缓冲 channel 接收

#### 直接接收

无缓冲 channel 在进行数据接收时，如果发送队列 sendq 已经有等待的发送 goroutine，那么当前 goroutine 会直接和这个发送者进行数据拷贝：

```Go
# go1.13:src/runtime/chan.go:473
if sg := c.sendq.dequeue(); sg != nil { 	//如果数据发送等待队列有正在等待的 goroutine
	// Found a waiting sender. If buffer is size 0, receive value
	// directly from sender. Otherwise, receive from head of queue
	// and add sender's value to the tail of the queue (both map to
	// the same buffer slot because the queue is full).
	recv(c, sg, ep, func() { unlock(&c.lock) }, 3) //数据接收处理
	return true, true
}
```

#### 接收阻塞

当发送队列 sendq 上没有正在等待发送数据的 goroutine 时，由于不存在缓冲队列，当前 goroutine 需要包装成上文提到的 ```sudog``` 结构体并插入到数据接收等待队列 recvq 中。之后便是调用 ```goparkunlock()``` 让当前 goroutine 进入漫长的等待了：

```Go
// no sender available: block on this channel.
gp := getg()
mysg := acquireSudog()
mysg.releasetime = 0
if t0 != 0 {
	mysg.releasetime = -1
}
// No stack splits between assigning elem and enqueuing mysg
// on gp.waiting where copystack can find it.
mysg.elem = ep
mysg.waitlink = nil
gp.waiting = mysg
mysg.g = gp
mysg.isSelect = false
mysg.c = c
gp.param = nil
c.recvq.enqueue(mysg)
goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)
```

和发送略微不同的是，由于数据接收操作并没有数据需要保护，所以并不需要类似 ```KeepAlive()``` 这样的接口来对数据进行安全锁定。

### 有缓冲 channel 接收

#### 直接接收

当 goroutine 从某个有缓冲 channel 接收数据时，依然会对发送队列 sendq 进行检查，如果队列中有等待发送的对象，那么表示当前有许多数据可以接收（满容量缓冲队列+发送等待队列），接收的 goroutine 会调用 ```recv()``` 接口进行数据交接。但是！请不要误会数据接收的对象是等待队列中的发送者。channel 的调度遵循 FIFO 原则，不会因为你是一个正在等待的 goroutine 而优先处理。然而具体是如何接收数据，在此先买个关子。

倘若发送队列 sendq 中没有等待的发送对象，不要慌，我们还有缓冲池。当缓冲队列大小不为0时，说明仍有数据可供接收，当前的 goroutine 会调用 ```typedmemmove()``` 将缓冲区的数据拷贝到自己的内存中：

```Go
# go1.13:src/runtime/chan.go:482
if c.qcount > 0 {	//判断循环队列中的缓冲对象
	// Receive directly from queue
	qp := chanbuf(c, c.recvx)
	if raceenabled {
		raceacquire(qp)
		racerelease(qp)
	}
	if ep != nil {
		typedmemmove(c.elemtype, ep, qp)	//直接将缓冲区中的数据拷贝到当前 goroutine 中
	}
	typedmemclr(c.elemtype, qp)	//清除缓冲区中的数据
	c.recvx++
	if c.recvx == c.dataqsiz {	//循环队列常规操作
		c.recvx = 0
	}
	c.qcount--
	unlock(&c.lock)
	return true, true
}
```

#### 接收阻塞

当缓冲队列中也没有可接收数据时，即使是一个有缓冲的 channel，goroutine 也要乖乖进入阻塞状态，这里的逻辑和上文中无缓冲 channel 接收阻塞逻辑是一样的，可以自行回顾[无缓冲 channel 接收数据阻塞过程](#接收阻塞)

### 唤醒

阻塞状态的接收 goroutine 在漫长的等待之后被唤醒，这个时候它会发现数据已经拷贝到自己的内存中（类似于睡了一天然后醒来发现一天的工资已经发到银行卡里那种感觉），需要自己做的仅仅是数据的整理检查等收尾工作：

```Go
# go1.13:src/runtime/chan.go:526
// someone woke us up
if mysg != gp.waiting {
	throw("G waiting list is corrupted")
}
gp.waiting = nil
if mysg.releasetime > 0 {
	blockevent(mysg.releasetime-t0, 2)
}
closed := gp.param == nil
gp.param = nil
mysg.c = nil
releaseSudog(mysg)
```

这里的检查相较于发送的唤醒操作少了一个 channel 的关闭状态检查。因为在设计上，即使 channel 已经被关闭，只要还有数据残留，channel 依然是可读的。

### How to receive ？

到目前为止，channel 中接收和发送的逻辑还是十分相似的，真正的区别在于 ```send()``` 和 ```recv()``` 的实现，也就是实际的数据传输操作。

在 ```recv()``` 函数中，程序只知道有数据需要接收，但不确定数据源是从 ```sudog``` 对象中还是从缓冲队列中获取，所以在接收数据前还需要根据 channel 的状态进行判断：

```Go
# go1.13:src/runtime/chan.go:554
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {	//无缓冲 channel 处理入口
		if raceenabled {
			racesync(c, sg)
		}
		if ep != nil {
			// copy data from sender
			recvDirect(c.elemtype, sg, ep)	//直接从发送的 goroutine 中拷贝数据
		}
	} else {	// 看注释↓↓↓↓↓↓
		// Queue is full. Take the item at the
		// head of the queue. Make the sender enqueue
		// its item at the tail of the queue. Since the
		// queue is full, those are both the same slot.
		qp := chanbuf(c, c.recvx)
		if raceenabled {	// 不用管系列
			raceacquire(qp)
			racerelease(qp)
			raceacquireg(sg.g, qp)
			racereleaseg(sg.g, qp)
		}
		// copy data from queue to receiver
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)	//将缓冲队列队首的数据拷贝到当前接收数据的 goroutine 中
		}
		// copy data from sender to queue
		typedmemmove(c.elemtype, qp, sg.elem)	//将阻塞的发送 goroutine 中的数据拷贝到缓冲队列中
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
```

在这里，传入的 sg 是发送数据方的一个已经进入休眠的 ```sudog``` 结构体。 首先检查缓冲队列大小，如果大小为0（无缓冲 channel ），调用 ```recvDirect()``` 对数据进行<font color=red>**直接拷贝**</font>。

如果是有缓冲队列的 channel，那么表示缓冲队列已经满员（因为发送队列已经有 goroutine 阻塞），接下来需要完成两件事：

1. 将缓冲队列队首的数据拷贝到当前接收数据的 goroutine 中。
2. 将阻塞的发送 goroutine 中的数据拷贝到缓冲队列队尾中。

完成了数据交接后，调用 ```goready()``` 轻轻唤醒阻塞的数据发送 goroutine，深藏功与名。

## 总结

- 许多文章将 channel 类比为“管道”之类的工具，但在了解 channel 的完整调度逻辑后，我更希望将其比作一个高速公路中的货物中转站。来来往往的货车司机（goroutine）在此装货卸货，有无仓库（缓冲）决定了货车司机需不需要在中转站逗留等待，迟到的货车司机负责货物的装载或拆卸（数据传输）并通知对应的司机完成货物交接。
- Go 基于 CSP 模型中的 process 和 channel 的概念设计了基于 channel 的并发模型并提倡“通过通信共享内存，而不是通过共享内存来通信”。然而这更多是一种行为上的概念，实际实现中还是无法避免地共享了 hchan 甚至缓冲 buf 等数据结构。唯一不同的是当两个 goroutine 直接交换数据时，他们是可以直接操作对方的内存空间（通过 ```sendDirect()``` 和 ```recvDirect()``` 两个方法），避免了数据的多次拷贝和额外的锁操作。
- 为了遵循 FIFO 原则，Go 语言用两个指针加数组实现了循环队列，实际上这大概也是队列的最佳实现方式了 ... ... 吧？
- channel 使用 mutex 来保障它是 goroutine 安全的，但是作为一个对性能要求极高的数据结构，我们可以发现解锁操作并不一定是在当前加锁的处理函数中。它通过传递解锁接口 ```unlockf()``` 来确保对数据修改结束后第一时间进行锁的释放，从而实现锁的覆盖时间是最短的。

## 参考资料

[diving-deep-into-the-golang-channels](https://speakerdeck.com/kavya719/understanding-channels)