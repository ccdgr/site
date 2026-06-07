---
date: '2024-03-08T08:55:00+08:00'
draft: false
title: 'Go 并发编程模式与示例'
tags: ["Go"]
description: "Go 并发实用模式：sync.WaitGroup、errgroup、sync.Once、sync.Pool、context、工作池、扇出扇入"
toc: false
---

> goroutine + channel 是 Go 并发的骨架，但真正的工程实践需要组合模式。

---

## sync.WaitGroup — 等待一组 goroutine

```go
func main() {
    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            fmt.Printf("worker %d done\n", id)
        }(i) // 注意传参，避免闭包捕获循环变量
    }

    wg.Wait()
    fmt.Println("all done")
}
```

**常见错误**：`wg.Add(1)` 放在 `go func` 里面，可能导致 Add 还没执行 Wait 就返回了。

## errgroup — 一组 goroutine 任一报错则取消

```go
import "golang.org/x/sync/errgroup"

func main() {
    g, ctx := errgroup.WithContext(context.Background())

    urls := []string{"https://a.com", "https://b.com", "https://c.com"}
    for _, url := range urls {
        url := url
        g.Go(func() error {
            resp, err := http.Get(url)
            if err != nil {
                return err
            }
            resp.Body.Close()
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        log.Fatal(err)
    }
}
```

`errgroup` 还支持限制并发数：`g.SetLimit(3)`。

## sync.Once — 只执行一次

```go
var (
    once sync.Once
    db   *sql.DB
)

func GetDB() *sql.DB {
    once.Do(func() {
        var err error
        db, err = sql.Open("mysql", dsn)
        if err != nil {
            panic(err)
        }
    })
    return db
}
```

## sync.Pool — 对象复用，降低 GC 压力

```go
var bufPool = sync.Pool{
    New: func() any {
        return make([]byte, 1024)
    },
}

func handler(w http.ResponseWriter, r *http.Request) {
    buf := bufPool.Get().([]byte)
    defer bufPool.Put(buf)

    // 使用 buf ...
    buf = buf[:0] // 重置
}
```

**注意**：Pool 中的对象随时可能被 GC 回收，不能假设 Put 进去就一定能 Get 到。

## context — 传递超时、取消、元数据

```go
func worker(ctx context.Context, id int) error {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("worker %d canceled: %v\n", id, ctx.Err())
            return ctx.Err()
        default:
            // do work
            time.Sleep(500 * time.Millisecond)
        }
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    for i := 0; i < 3; i++ {
        go worker(ctx, i)
    }

    <-ctx.Done()
    fmt.Println("main exit:", ctx.Err())
}
```

```
context.Background()
  └── WithCancel()        → 手动取消
  └── WithTimeout()       → 超时自动取消
  └── WithDeadline()      → 到时间点自动取消
  └── WithValue()         → 携带 key-value（少用，适合 traceID 等横切关注点）
```

## 工作池（Worker Pool）

```go
func workerPool() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

    // 启动 3 个 worker
    var wg sync.WaitGroup
    for w := 1; w <= 3; w++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := range jobs {
                results <- j * 2
            }
        }(w)
    }

    // 发送任务
    for j := 1; j <= 10; j++ {
        jobs <- j
    }
    close(jobs)

    // 等待 worker 完成，关闭结果通道
    go func() {
        wg.Wait()
        close(results)
    }()

    // 收集结果
    for r := range results {
        fmt.Println(r)
    }
}
```

## 扇出扇入（Fan-out / Fan-in）

```go
// 扇出：一个输入 channel 分发给多个 worker
func fanOut(in <-chan int, workers int) []<-chan int {
    channels := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        ch := make(chan int)
        go func(out chan int) {
            defer close(out)
            for v := range in {
                out <- v * v
            }
        }(ch)
        channels[i] = ch
    }
    return channels
}

// 扇入：多个 channel 合并到一个
func fanIn(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}

func main() {
    in := make(chan int)
    go func() {
        for i := 1; i <= 10; i++ {
            in <- i
        }
        close(in)
    }()

    workers := fanOut(in, 4)
    out := fanIn(workers...)
    for v := range out {
        fmt.Println(v)
    }
}
```

## Mutex vs Channel 的选择

> **规则**：如果你的 goroutine 是在「管理状态」，用 mutex；如果是在「协调通信」，用 channel。

```go
// 状态管理 → sync.Mutex
type Counter struct {
    mu    sync.Mutex
    count int
}
func (c *Counter) Inc() { c.mu.Lock(); c.count++; c.mu.Unlock() }

// 协调通信 → channel
done := make(chan struct{})
go func() { /* work */ close(done) }()
<-done
```

## 总结

| 模式 | 场景 |
|---|---|
| WaitGroup | 等待多个 goroutine 完成 |
| errgroup | 任意错误即取消，限制并发 |
| sync.Once | 单例、初始化 |
| sync.Pool | 对象复用，降低 GC |
| context | 超时、取消、值传递 |
| 工作池 | 限制并发、任务队列 |
| 扇出扇入 | 并行处理 + 聚合 |
