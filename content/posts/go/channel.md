---
date: '2024-01-22T16:30:00+08:00'
draft: false
title: 'Go Channel 底层原理与使用'
tags: ["Go"]
description: "Go channel 底层结构、hchan 环形队列、阻塞与唤醒、select 实现、常用模式"
toc: false
---

> 「不要通过共享内存来通信，而要通过通信来共享内存。」—— Go 并发哲学。

---

## 底层结构 hchan

```go
// runtime/chan.go 中的 hchan 结构（简化）
type hchan struct {
    qcount   uint           // 队列中元素个数
    dataqsiz uint           // 环形队列容量
    buf      unsafe.Pointer // 指向环形队列的指针
    elemsize uint16         // 元素大小
    closed   uint32         // 是否关闭
    sendx    uint           // 发送索引
    recvx    uint           // 接收索引
    recvq    waitq          // 等待接收的 goroutine 队列
    sendq    waitq          // 等待发送的 goroutine 队列
    lock     mutex          // 互斥锁
}
```

```
无缓冲 channel                 有缓冲 channel（容量=4）
┌───────────────────┐          ┌──────────────────────┐
│ 发送者 → ← 接收者  │          │ ┌──┬──┬──┬──┐       │
│                   │          │ │  │  │  │  │  buf   │
│  sendq    recvq   │          │ └──┴──┴──┴──┘       │
│   队列     队列    │          │ sendx=1  recvx=0     │
└───────────────────┘          │ sendq        recvq   │
                                └──────────────────────┘
```

## 无缓冲 vs 有缓冲

```go
ch1 := make(chan int)      // 无缓冲：发送和接收必须同时就绪
ch2 := make(chan int, 10)  // 有缓冲：缓冲满时发送阻塞，空时接收阻塞
```

| | 无缓冲 | 有缓冲 |
|---|---|---|
| 发送 | 等待接收者就绪 | 缓冲有空位即可，满则阻塞 |
| 接收 | 等待发送者就绪 | 缓冲有数据即可，空则阻塞 |
| 用途 | 同步、通知 | 削峰填谷、解耦生产消费 |

## 发送和接收发生了什么

### 发送 `ch <- v`

```
1. 加锁
2. 如果 recvq 有等待的接收者 → 直接把值拷贝给接收者，解锁，唤醒接收者
3. 如果 buf 有空位 → 数据入队，sendx++，解锁
4. 如果 buf 满了 → 当前 G 入 sendq 等待队列，解锁，挂起
```

### 接收 `v := <-ch`

```
1. 加锁
2. 如果 sendq 有等待的发送者（无缓冲或有缓冲且已空）
   → 从发送者直接取数据（或从 buf 取），解锁，唤醒发送者
3. 如果 buf 有数据 → 从 buf 取数据，recvx++，解锁
4. 如果 buf 空的 → 当前 G 入 recvq 等待队列，解锁，挂起
```

### 关闭 `close(ch)`

```
1. 加锁
2. 设置 closed=1
3. 唤醒 recvq 中所有等待的 G（它们会收到零值 + ok=false）
4. 唤醒 sendq 中所有等待的 G（它们会 panic）
5. 解锁
```

## select 实现

```go
select {
case v := <-ch1:
    // ...
case ch2 <- x:
    // ...
default:
    // ...
}
```

runtime 将 select 中所有的 case 放入一个数组，**伪随机打乱顺序**后逐一检查：

1. 如果有 case 就绪 → 执行它
2. 如果没有就绪但有 default → 执行 default
3. 如果都没有 → 当前 G 挂到所有 channel 的等待队列，等待唤醒

**随机打乱保证了公平性**，避免前面的 case 一直饿死后面的。

## 常见模式

### 生产者-消费者

```go
func producer(ch chan<- int) {
    for i := 0; i < 10; i++ {
        ch <- i
    }
    close(ch)
}

func consumer(ch <-chan int) {
    for v := range ch {  // channel 关闭后 range 自动退出
        fmt.Println(v)
    }
}
```

### 通知/信号

```go
done := make(chan struct{})
go func() {
    // do work
    close(done)
}()
<-done  // 等待完成
```

### 控制并发数

```go
sem := make(chan struct{}, 5)  // 最多 5 个并发
for _, url := range urls {
    sem <- struct{}{}
    go func(u string) {
        defer func() { <-sem }()
        fetch(u)
    }(url)
}
// 等待所有完成
for i := 0; i < 5; i++ {
    sem <- struct{}{}
}
```

### 超时控制

```go
select {
case v := <-ch:
    fmt.Println("received:", v)
case <-time.After(3 * time.Second):
    fmt.Println("timeout")
}
```

### 定时器

```go
ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()
for range ticker.C {
    fmt.Println("tick")
}
```

## 注意事项

- **向已关闭 channel 发送 → panic**
- **重复关闭 channel → panic**
- **从已关闭 channel 接收 → 返回零值，不会 panic**
- **nil channel 发送/接收 → 永久阻塞**
- **最好由发送方关闭 channel**，不要由接收方关闭

```go
// 安全关闭
func safeClose(ch chan int) (closed bool) {
    defer func() {
        if recover() != nil {
            closed = false
        }
    }()
    close(ch)
    return true
}
```

## 总结

- channel 底层是环形队列 + 等待队列 + 互斥锁
- 无缓冲用于同步，有缓冲用于解耦
- select 随机打乱 case 顺序，保证公平
- 发送方关闭 channel，range 自动退出
- 用 channel 替代 mutex 传递数据所有权
