---
title: "八股文 Go篇"
date: 2022-03-06T22:50:24+08:00
draft: false
tags: ["面试"]
---
结合一些实际面试遇到的问题和网上看的八股文，总结一下面试里常问到的 Go 相关的问题。
<!--more-->

## 并发编程相关
1）`sync.pool` 用过吗，了解的话讲一下
> 得物

主要是为了保存和复用一些临时对象，避免频繁的内存分配，减少 GC。光看这句话可能不直观，看下这个例子就清楚了:
```golang
var studentPool = sync.Pool{
    New: func() interface{} { 
        return new(Student) 
    },
}

func BenchmarkUnmarshal(b *testing.B) {
	for n := 0; n < b.N; n++ {
		stu := &Student{}
		json.Unmarshal(buf, stu)
	}
}

func BenchmarkUnmarshalWithPool(b *testing.B) {
	for n := 0; n < b.N; n++ {
		stu := studentPool.Get().(*Student)
		json.Unmarshal(buf, stu)
		studentPool.Put(stu)
	}
}
```

2）go 里面互斥锁和读写锁有什么区别
> 得物、高频、必会

普通互斥锁读写都会阻塞，读写锁在纯读操作的时候不会阻塞，写操作之间互斥，读写之间也会互斥

3）使用互斥锁的时候会不会出现某个 goroutine 一直取不到锁的情况？互斥锁有哪些状态？如何保证公平？
> 得物

不会，互斥锁有正常状态和饥饿状态两种状态。

正常状态下，所有等待锁的 goroutine 会按 FIFO 的顺序等待，等待队列中被唤醒的 goroutine 不会立刻持有锁，而是会和最新请求锁的 goroutine 竞争，不过因为新请求的 goroutine 正在占用 cpu，而且可能有很多，所以一般都竞争不过，这时候最新请求的 goroutine 会拿到锁，被唤醒的 goroutine 会回到队首继续等待。

如果队首的等待超过 1ms，那么就会进入饥饿模式，饥饿模式下最新来的 goroutine 不会和等待队列里的竞争，总是优先把锁给等待队列里的第一个。

当等待队列里的 goroutine 获取了锁并且它在队列末尾时，或者某个等待 goroutine 等待时间小于 1ms 时，那么互斥锁的状态会回到正常模式，如此往复。

4）GPM 调度机制
> 字节、b站、shopee、高频、必会
1. 把 goroutine（G）以 m:n 的形式通过 processor 队列（P）调度映射到操作系统线程（M）上
2. M 会优先从绑定的 P 队列上去 G 执行，如果取不到会去全局队列取，如果还是取不到会去其它 M 的 P 队列上偷一半
3. 当发生系统调用（比如 gc）阻塞时，M 会解绑 P 交给其他 M，让 P 中剩余的 G 不至于空等
4. 当发生用户态（比如网络 IO）阻塞时，G 会进入等待队列，M 跳过它继续执行后面的 G
5. 当 G 被某个 G2 唤醒时，G 会首先尝试加入 G2 所在的 P 队列，如果满了的话则会加入全局队列

5）`GOMAXPROCS` 环境变量的作用
> b 站
1. 这个环境变量为 Go 程序设置了逻辑 CPU 的数目，也就是 M 的数目
2. 合理的设置这个值，比如计算素数这种 CPU 密集型应用里可以有效提升并发性能，如果不合理，会导致程序非常占用系统资源
3. go 1.5 版本之前这个值默认是 1，1.5 版本之后默认是机器 CPU 核数

## 数据结构相关
1）`sync.map` 的实现原理
> 字节、高频、必会
1. 通过 read 和 dirty 两个字段来进行读写分离，最新写入数据始终写进 dirty，然后定期同步到 read 上
2. 读的时候先从 read 读，如果读不到则穿透到 dirty 上读
3. 读取 read 时不需要加锁，但是读写 dirty 时都要加锁
4. 通过 misses 字段来统计 read 被穿透的次数，超过一定量时将 dirty 数据同步到 read 上
5. 删除数据时通过标记来延迟删除