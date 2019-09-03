---
layout: post
title: 'Go Channel'
subtitle: '发送与接收流程'
date: 2019-08-30
categories: 技术
cover: ''
tags: Golang
---

环境：
```go
$ go version
go version go1.12.7 linux/amd64
$ uname -a
18.04.1-Ubuntu SMP x86_64 x86_64 x86_64 GNU/Linux
```

## 大致流程
___

Go runtime包里的[chan.go](https://github.com/golang/go/blob/master/src/runtime/chan.go)实现里channel的大部分功能。其中`chansend()`和`chanrecv()`两个函数完成了channel的发送与接收功能。

值得注意的是，源码中没有特意的将buffered channel与unbuffered channel分开处理。二者的发送与接收仍然是通过`chansend()`和`chanrecv()`两个函数完成。二者处理逻辑大致相同。

[func chansend()](https://github.com/golang/go/blob/master/src/runtime/chan.go#L142)

**c <- *ep**

- 向nil channel发送，g泄漏
- 向closed channel发送，panic
- 当hchan.buf为空，且hchan.recvq不为空时，调用send()函数，直接将*ep复制到hchan.recvq.elem
- 当hchan.buf不为空，此时hchan.recvq肯定为空，直接向hchan.buf中发送
- 当hchan.buf满了，此时hchan.recvq肯定为空，park getg()...
- take away from sudog.elem
- ready...

[func chanrecv()](https://github.com/golang/go/blob/master/src/runtime/chan.go#L421)

***ep <- c**

- 从nil channel接收，g泄漏
- 从closed channel接收，接收零值
- 当hchan.buf满了，且hchan.sendq不为空时，调用revc()函数，先从hchan.buf中获取一个，然后将hchan.sendq.elem拷贝进hchan.buf
- 当hchan.buf没满，此时hchan.sendq肯定为空，直接从hchan.buf中获取
- 当hchan.buf为空，此时hchan.sendq肯定为空，park getg()...
- send to sudog.elem
- ready...

两个函数的代码逻辑并不复杂，两个函数中都出现了g的'泄漏'，值得理解一下，我不知道使用'泄漏'一词是否准确。之所以称之为'泄漏'，且看下面的代码：

```go
if c == nil {
	if !block {
        ...
	}
	gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
	throw("unreachable")
}
```
当channel为nil时，直接将当前g park，重新调度。这会让我们再也无法重新调度到当前g。与之类似的操作：
```go
gp := getg()
mysg := acquireSudog()
...
mysg.g = gp
...
c.sendq.enqueue(mysg)
goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)
```
以这种方式park一个g，在以后的某个时刻，可以通过hchan.sendq拿到g的指针，让其重新进入调度。

## 与Kernel设计再次"撞脸"
___

对channel的整个流程分析下来，发现其在发送时的优化与Kernel中System V 消息队列的发送优化类似：

```c
if (!pipelined_send(msq, msg)) { /* 如果有进程在等待消息，直接给它 */
		/* no one is waiting for this message, enqueue it */
		list_add_tail(&msg->m_list, &msq->q_messages);
		msq->q_cbytes += msgsz;
		msq->q_qnum++;
		atomic_add(msgsz, &ns->msg_bytes);
		atomic_inc(&ns->msg_hdrs);
	}
```
详见[Kernel源码](https://elixir.bootlin.com/linux/v3.2.44/source/ipc/msg.c#L706)

## 未解疑惑
___

最后还有一点关于创建channel的疑惑：
```go
/*
	hchan.buf为空: unbuffered channel
*/
case mem == 0:
	// Queue or element size is zero.
	c = (*hchan)(mallocgc(hchanSize, nil, true))
	// Race detector uses this location for synchronization.
	c.buf = c.raceaddr()
case elem.kind&kindNoPointers != 0:
	// Elements do not contain pointers.
	// Allocate hchan and buf in one call.
	c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
	c.buf = add(unsafe.Pointer(c), hchanSize)
default:
	// Elements contain pointers.
	c = new(hchan)
	c.buf = mallocgc(mem, elem, true)
}
```
如果hchan.buf中包含指针，则不将两段内存（hchan与hchan.buf）连续分配，为什么？TODO

参考资料：
- [src/runtime/chan.go](https://golang.org/src/runtime/chan.go)